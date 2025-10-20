<script lang="ts">
  import type { Moment } from "moment";
  import type { LocalTask } from "../../task-types";
  import { get } from "svelte/store";
  import { isNotVoid } from "typed-assert";

  import { getObsidianContext } from "../../context/obsidian-context";
  import { isToday } from "../../global-store/current-time";
  import { getVisibleHours, snap } from "../../global-store/derived-settings";
  import {
    getIsomorphicClientY,
    isTouchEvent,
    offsetYToMinutes,
  } from "../../util/dom";
  import { minutesToMomentOfDay } from "../../util/moment";
  import { getRenderKey } from "../../util/task-utils";
  import { createGestures } from "../actions/gestures";
  import { createTimeBlockMenu } from "../time-block-menu";

  import Column from "./column.svelte";
  import LocalTimeBlock from "./local-time-block.svelte";
  import Needle from "./needle.svelte";
  import PositionedTimeBlock from "./positioned-time-block.svelte";
  import UnscheduledTimeBlock from "./unscheduled-time-block.svelte";

  const {
    day,
    isUnderCursor = false,
  }: { day: Moment; isUnderCursor?: boolean } = $props();

  const {
    settings,
    editContext: {
      confirmEdit,
      handlers: { handleContainerMouseDown },
      getDisplayedTasksForTimeline,
      editOperation,
    },
    pointerDateTime,
    getDisplayedTasksWithClocksForTimeline,
    settingsSignal,
    workspaceFacade,
  } = getObsidianContext();

  type LocalTaskWithLocation = LocalTask & {
    location: {
      path: string;
      position: { start: { line: number } };
    };
  };

  function hasTaskLocation<T extends LocalTask>(
    task: T,
  ): task is T & LocalTaskWithLocation {
    const location = task.location as
      | LocalTaskWithLocation["location"]
      | undefined;

    return Boolean(
      location &&
        typeof location.path === "string" &&
        typeof location.position?.start?.line === "number",
    );
  }

  function revealTaskInFile(task: LocalTask) {
    if (!hasTaskLocation(task)) {
      return;
    }

    const {
      location: {
        path,
        position: {
          start: { line },
        },
      },
    } = task;

    void workspaceFacade.revealLineInFile(path, line);
  }

  function showTimeBlockMenu(event: MouseEvent, task: LocalTask) {
    if (!hasTaskLocation(task)) {
      return;
    }

    event.preventDefault();

    createTimeBlockMenu({ event, task, workspaceFacade });
  }

  const displayedTasksForTimeline = $derived(getDisplayedTasksForTimeline(day));
  const displayedTasksWithClocksForTimeline = $derived(
    getDisplayedTasksWithClocksForTimeline(day),
  );

  let el: HTMLElement | undefined = $state();

  function updatePointerDateTime(event: MouseEvent | TouchEvent) {
    isNotVoid(el);

    const viewportToElOffsetY = el.getBoundingClientRect().top;
    const borderTopToPointerOffsetY =
      getIsomorphicClientY(event) - viewportToElOffsetY;
    const newOffsetY = snap(borderTopToPointerOffsetY, $settings);

    const minutesSinceMidnight = offsetYToMinutes(
      newOffsetY,
      settingsSignal.current.zoomLevel,
      settingsSignal.current.startHour,
    );
    const dateTime = minutesToMomentOfDay(
      minutesSinceMidnight,
      window.moment(day),
    );
    // todo: might hurt perf. Need to check for identity of time not to run on every change of coords
    pointerDateTime.set({ dateTime, type: "dateTime" });
  }

  function handleContainerPointerDown(event: MouseEvent | TouchEvent) {
    updatePointerDateTime(event);
    handleContainerMouseDown();
  }

  const timelineGestures = createGestures({
    onlongpress: (event) => {
      if (event.target !== el) {
        return;
      }

      handleContainerPointerDown(event);
    },
    onpanmove: (event) => {
      if (get(editOperation)) {
        updatePointerDateTime(event);
      }
    },
    onpanend: confirmEdit,
    options: { mouseSupport: false },
  });
</script>

{#if $settings.timelineColumns.planner}
  <Column visibleHours={getVisibleHours($settings)}>
    {#if $isToday(day)}
      <Needle autoScrollBlocked={isUnderCursor} />
    {/if}

    <div
      bind:this={el}
      class="tasks absolute-stretch-x"
      onpointerdown={(event) => {
        if (isTouchEvent(event) || event.target !== el) {
          return;
        }

        handleContainerPointerDown(event);
      }}
      onpointermove={updatePointerDateTime}
      onpointerup={confirmEdit}
      use:timelineGestures
    >
      {#each $displayedTasksForTimeline.withTime as task (getRenderKey(task))}
        <PositionedTimeBlock {task}>
          <UnscheduledTimeBlock {task} />
        </PositionedTimeBlock>
      {/each}
    </div>
  </Column>
{/if}

{#if $settings.timelineColumns.timeTracker}
  <Column
    --column-background-color="hsl(var(--color-accent-hsl), 0.03)"
    visibleHours={getVisibleHours($settings)}
  >
    {#if $isToday(day)}
      <Needle autoScrollBlocked={isUnderCursor} showBall={false} />
    {/if}

    <div class="tasks absolute-stretch-x">
      {#each $displayedTasksWithClocksForTimeline as task (getRenderKey(task))}
        <PositionedTimeBlock {task}>
          <LocalTimeBlock
            ondblclick={() => {
              revealTaskInFile(task);
            }}
            oncontextmenu={(event) => {
              showTimeBlockMenu(event, task);
            }}
            {task}
          />
        </PositionedTimeBlock>
      {/each}
    </div>
  </Column>
{/if}

<style>
  .tasks {
    top: 0;
    bottom: 0;

    display: flex;
    flex-direction: column;

    margin-inline: var(--size-4-2);
  }
</style>
