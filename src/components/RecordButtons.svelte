<script lang="ts">
  export let startRecording: () => void;
  export let stopRecording: () => void;
  export let isRecording: boolean;
</script>

<div class="flex justify-center mb-8 gap-6">
  <!-- svelte-ignore a11y_consider_explicit_label -->
  <button
    on:click={startRecording}
    disabled={isRecording}
    class="w-16 h-16 rounded-full bg-white shadow-lg flex items-center justify-center focus:outline-none hover:bg-gray-50 disabled:opacity-50 disabled:cursor-not-allowed"
    title="سجل"
  >
    <svg viewBox="0 0 100 100" class="w-10 h-10">
      <circle cx="50" cy="50" r="46" fill={isRecording ? "#f87171" : "#ef4444"}
      ></circle>
    </svg>
  </button>

  <!-- svelte-ignore a11y_consider_explicit_label -->
  <button
    on:click={stopRecording}
    disabled={!isRecording}
    class="w-16 h-16 rounded-full bg-white shadow-lg flex items-center justify-center focus:outline-none hover:bg-gray-50 disabled:opacity-50 disabled:cursor-not-allowed"
    title="قف"
  >
    <svg viewBox="0 0 100 100" class="w-10 h-10">
      <path d="M12 12h76v76H12z" fill="#1f2937"></path>
    </svg>
  </button>
</div>
