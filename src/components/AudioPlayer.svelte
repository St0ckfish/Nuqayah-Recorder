<script lang="ts">
  export let currentRecording: any;
  export let seekerElement: HTMLDivElement;
  export let updateCurrentTime: (e: MouseEvent) => void;
  export let duration: number;
  export let currentTime: number;
  export let formatTime: (seconds: number) => string;
  export let togglePlay: () => Promise<void> | void;
  export let isPlaying: boolean;
  export let toggleRepeat: () => void;
  export let repeat: boolean;
  export let playbackRate: number;
  export let changePlaybackRate: (increment: number) => void;
  export let skip: (seconds: number) => void;
  export let downloadRecording: (recording: any) => void;
</script>

<div class="bg-white shadow-lg rounded-lg p-4 mb-8 { !currentRecording ? 'opacity-50 pointer-events-none' : '' }">
  {#if currentRecording}
    <div class="flex justify-between items-center mb-2">
      <button
        on:click={() => downloadRecording(currentRecording)}
        class="text-blue-600 hover:text-blue-800 cursor-pointer focus:outline-none flex items-center"
        title="تنزيل التسجيل"
      >
        <svg
          class="h-5 w-5"
          fill="none"
          stroke="currentColor"
          viewBox="0 0 24 24"
        >
          <path
            stroke-linecap="round"
            stroke-linejoin="round"
            stroke-width="2"
            d="M4 16v1a3 3 0 003 3h10a3 3 0 003-3v-1m-4-4l-4 4m0 0l-4-4m4 4V4"
          />
        </svg>
        <span class="mr-1">تنزيل</span>
      </button>
      <div class="text-lg font-medium text-center text-black">
        {currentRecording.name}
      </div>
      <div class="w-14"></div>
    </div>
  {/if}

  <!-- Seeker -->
  <!-- svelte-ignore a11y_click_events_have_key_events -->
  <!-- svelte-ignore a11y_no_static_element_interactions -->
  <div
    dir="ltr"
    bind:this={seekerElement}
    on:click={updateCurrentTime}
    class="h-3 bg-gray-200 rounded-full mb-3 cursor-pointer overflow-hidden"
  >
    <div
      class="h-full bg-blue-500 rounded-full"
      style="width: {duration > 0 ? (currentTime / duration) * 100 : 0}%"
    ></div>
  </div>

  <!-- Controls -->
  <div class="flex justify-between items-center mb-3">
    <div class="text-gray-600">{formatTime(currentTime)}</div>

    <div class="flex items-center gap-2">
      <button
        on:click={togglePlay}
        class="w-16 h-16 cursor-pointer rounded-full bg-blue-500 text-white flex items-center justify-center focus:outline-none hover:bg-blue-600"
      >
        {#if isPlaying}
          <svg
            class="h-8 w-8"
            viewBox="0 0 24 24"
            fill="none"
            stroke="currentColor"
            stroke-width="2"
            stroke-linecap="round"
            stroke-linejoin="round"
          >
            <rect x="6" y="4" width="4" height="16" />
            <rect x="14" y="4" width="4" height="16" />
          </svg>
        {:else}
          <svg
            class="h-8 w-8"
            viewBox="0 0 24 24"
            fill="none"
            stroke="currentColor"
            stroke-width="2"
            stroke-linecap="round"
            stroke-linejoin="round"
          >
            <polygon points="5 3 19 12 5 21 5 3" />
          </svg>
        {/if}
      </button>

      <!-- svelte-ignore a11y_consider_explicit_label -->
      <button
        on:click={toggleRepeat}
        class="w-16 h-16 rounded-full {repeat ? 'bg-blue-100 text-blue-600' : 'bg-gray-100 text-gray-600'} flex items-center justify-center focus:outline-none hover:bg-blue-100"
      >
        <svg
          class="h-8 w-8"
          width="24"
          height="24"
          viewBox="0 0 24 24"
          stroke-width="2"
          stroke="currentColor"
          fill="none"
          stroke-linecap="round"
          stroke-linejoin="round"
        >
          <path stroke="none" d="M0 0h24v24H0z" />
          <path d="M4 12v-3a3 3 0 0 1 3 -3h13m-3 -3l3 3l-3 3" />
          <path d="M20 12v3a3 3 0 0 1 -3 3h-13m3 3l-3-3l3-3" />
        </svg>
      </button>
    </div>

    <div class="text-gray-600">-{formatTime(duration - currentTime)}</div>
  </div>

  <div class="flex justify-between items-center">
    <!-- Speed controls -->
    <div class="flex items-center gap-1">
      <!-- svelte-ignore a11y_consider_explicit_label -->
      <button
        on:click={() => changePlaybackRate(-0.1)}
        class="w-8 h-8 rounded-full bg-gray-100 text-black flex items-center justify-center focus:outline-none hover:bg-gray-200"
      >
        <svg
          class="w-4 h-4"
          fill="none"
          stroke="currentColor"
          viewBox="0 0 24 24"
        >
          <path
            stroke-linecap="round"
            stroke-linejoin="round"
            stroke-width="2"
            d="M15 19l-7-7 7-7"
          ></path>
        </svg>
      </button>

      <div class="px-2 py-1 text-black text-sm bg-gray-100 rounded-md min-w-16 text-center">
        {playbackRate.toFixed(2)}x
      </div>

      <!-- svelte-ignore a11y_consider_explicit_label -->
      <button
        on:click={() => changePlaybackRate(0.1)}
        class="w-8 h-8 rounded-full bg-gray-100 text-black flex items-center justify-center focus:outline-none hover:bg-gray-200"
      >
        <svg
          class="w-4 h-4"
          fill="none"
          stroke="currentColor"
          viewBox="0 0 24 24"
        >
          <path
            stroke-linecap="round"
            stroke-linejoin="round"
            stroke-width="2"
            d="M9 5l7 7-7 7"
          ></path>
        </svg>
      </button>
    </div>

    <!-- Skip controls -->
    <div class="flex items-center gap-2">
      <button
        on:click={() => skip(15)}
        class="w-8 h-8 rounded-full bg-gray-100 text-black flex items-center justify-center focus:outline-none hover:bg-gray-200"
      >
        <svg
          class="w-4 h-4"
          fill="none"
          stroke="currentColor"
          viewBox="0 0 24 24"
        >
          <path
            stroke-linecap="round"
            stroke-linejoin="round"
            stroke-width="2"
            d="M11.933 12.8a1 1 0 000-1.6L6.6 7.2A1 1 0 005 8v8a1 1 0 001.6.8l5.333-4zM19.933 12.8a1 1 0 000-1.6l-5.333-4A1 1 0 0013 8v8a1 1 0 001.6.8l5.333-4z"
          ></path>
        </svg>
        <span class="text-xs">15</span>
      </button>

      <button
        on:click={() => skip(-15)}
        class="w-8 h-8 rounded-full bg-gray-100 text-black flex items-center justify-center focus:outline-none hover:bg-gray-200"
      >
        <span class="text-xs">15</span>
        <svg
          class="w-4 h-4"
          fill="none"
          stroke="currentColor"
          viewBox="0 0 24 24"
        >
          <path
            stroke-linecap="round"
            stroke-linejoin="round"
            stroke-width="2"
            d="M12.066 11.2a1 1 0 000 1.6l5.334 4A1 1 0 0019 16V8a1 1 0 00-1.6-.8l-5.333 4zM4.066 11.2a1 1 0 000 1.6l5.334 4A1 1 0 0011 16V8a1 1 0 00-1.6-.8l-5.334 4z"
          ></path>
        </svg>
      </button>
    </div>
  </div>
</div>
