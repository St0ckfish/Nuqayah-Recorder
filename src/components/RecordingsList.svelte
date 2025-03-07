<script lang="ts">
  export let loading: boolean;
  export let recordings: any[];
  export let currentRecording: any;
  export let setCurrentRecording: (recording: any) => void;
  export let renameRecording: (recording: any, newName: string) => void;
  export let deleteRecording: (recording: any) => void;
  export let formatTime: (seconds: number) => string;
</script>

<div class="bg-white shadow-lg rounded-lg p-4 mb-8">
  <h2 class="text-xl font-bold mb-4">التسجيلات السابقة</h2>

  {#if loading}
    <div class="text-center py-4">جاري التحميل...</div>
  {:else if recordings.length === 0}
    <div class="text-center py-4 text-gray-500">لا توجد تسجيلات سابقة</div>
  {:else}
    <ul class="divide-y divide-gray-200">
      {#each recordings as recording (recording.id)}
        <li class="py-3 flex items-center justify-between">
          <div class="flex-1 min-w-0">
            <button
              on:click={() => setCurrentRecording(recording)}
              class="block w-full text-left cursor-pointer focus:outline-none {currentRecording?.id ===
              recording.id
                ? 'text-blue-600 font-medium'
                : 'text-black'}"
            >
              <div class="text-base">{recording.name}</div>
              <div
                class="text-sm text-gray-500 flex items-center justify-between"
              >
                <span
                  >{new Date(recording.date).toLocaleDateString("ar-SA")}</span
                >
                <span class="ml-2">{formatTime(recording.duration)}</span>
              </div>
            </button>
          </div>

          <div class="flex items-center gap-2 mr-2">
            <!-- svelte-ignore a11y_consider_explicit_label -->
            <button
              on:click={() => {
                const newName = prompt(
                  "أدخل اسمًا جديدًا للتسجيل:",
                  recording.name
                );
                if (newName && newName.trim()) {
                  renameRecording(recording, newName.trim());
                }
              }}
              class="text-gray-500 hover:text-gray-700 focus:outline-none"
              title="تغيير الاسم"
            >
              <svg
                class="w-5 h-5"
                fill="none"
                stroke="currentColor"
                viewBox="0 0 24 24"
              >
                <path
                  stroke-linecap="round"
                  stroke-linejoin="round"
                  stroke-width="2"
                  d="M11 5H6a2 2 0 00-2 2v11a2 2 0 002 2h11a2 2 0 002-2v-5m-1.414-9.414a2 2 0 112.828 2.828L11.828 15H9v-2.828l8.586-8.586z"
                ></path>
              </svg>
            </button>

            <!-- svelte-ignore a11y_consider_explicit_label -->
            <button
              on:click={() => {
                if (confirm("هل أنت متأكد من حذف هذا التسجيل؟")) {
                  deleteRecording(recording);
                }
              }}
              class="text-gray-500 hover:text-red-600 focus:outline-none"
              title="حذف"
            >
              <svg
                class="w-5 h-5"
                fill="none"
                stroke="currentColor"
                viewBox="0 0 24 24"
              >
                <path
                  stroke-linecap="round"
                  stroke-linejoin="round"
                  stroke-width="2"
                  d="M19 7l-.867 12.142A2 2 0 0116.138 21H7.862a2 2 0 01-1.995-1.858L5 7m5 4v6m4-6v6m1-10V4a1 1 0 00-1-1h-4a1 1 0 00-1 1v3M4 7h16"
                ></path>
              </svg>
            </button>
          </div>
        </li>
      {/each}
    </ul>
  {/if}
</div>
