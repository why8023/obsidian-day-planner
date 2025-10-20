<script lang="ts">
  import type { Moment } from "moment";
  import { get } from "svelte/store";
  import { isNotVoid } from "typed-assert";

  import { getObsidianContext } from "../../context/obsidian-context";
  import { isToday } from "../../global-store/current-time";
  import { getVisibleHours, snap } from "../../global-store/derived-settings";
  import type { LocalTask, WithPlacing } from "../../task-types";
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

  function revealTaskInFile(task: WithPlacing<LocalTask>) {
    const location = task.location;
    const line = location?.position?.start?.line;

    if (!location || typeof line !== "number") {
      return;
    }

    void workspaceFacade.revealLineInFile(location.path, line);
  }

  function handleClockTaskContextMenu(
    event: Event,
    task: WithPlacing<LocalTask>,
  ) {
    const { location } = task;

    if (!location) {
      return;
    }

    if (!(event instanceof MouseEvent)) {
      return;
    }

    event.preventDefault();
    event.stopPropagation();

    createTimeBlockMenu({ event, task, workspaceFacade });
  }
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
            {task}
            on:contextmenu={(event) => handleClockTaskContextMenu(event, task)}
            on:dblclick={() => revealTaskInFile(task)}
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
