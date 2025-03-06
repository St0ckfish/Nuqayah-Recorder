<script lang="ts">
  import { onMount, onDestroy } from "svelte";
  import { get, set, del, keys } from "idb-keyval";

  // Types
  type Recording = {
    id: string;
    blob: Blob;
    date: Date;
    url: string;
    name: string;
    duration: number; // Added duration field to the schema
  };

  // State
  let audio: HTMLAudioElement;
  let recorder: MediaRecorder | null = null;
  let recordedBlob: Blob | null = null;
  let isRecording = false;
  let isPaused = false;
  let isPlaying = false;
  let recordings: Recording[] = [];
  let currentRecording: Recording | null = null;
  let seekerWidth = 0;
  let seekerElement: HTMLDivElement;
  let playbackRate = 1;
  let duration = 0;
  let currentTime = 0;
  let repeat = false;
  let loading = true;
  let recordingStartTime: number | null = null; // Track when recording started

  // Setup on component mount
  onMount(async () => {
    audio = new Audio();
    setupAudioListeners();
    await loadRecordings();
    loading = false;
  });

  // Clean up on component destroy
  onDestroy(() => {
    if (recorder && isRecording) {
      stopRecording();
    }

    if (audio) {
      audio.pause();
      audio.removeEventListener("play", handlePlay);
      audio.removeEventListener("pause", handlePause);
      audio.removeEventListener("timeupdate", handleTimeUpdate);
      audio.removeEventListener("ended", handleEnded);
      audio.removeEventListener("loadedmetadata", handleLoadedMetadata);
    }

    // Revoke URLs for recordings
    recordings.forEach((recording) => {
      URL.revokeObjectURL(recording.url);
    });
  });

  // Setup audio event listeners
  function setupAudioListeners() {
    audio.addEventListener("play", handlePlay);
    audio.addEventListener("pause", handlePause);
    audio.addEventListener("timeupdate", handleTimeUpdate);
    audio.addEventListener("ended", handleEnded);
    audio.addEventListener("loadedmetadata", handleLoadedMetadata);
  }

  // Load recordings from IndexedDB
  async function loadRecordings() {
    try {
      const recordingKeys = await keys();
      const recordingPromises = recordingKeys
        .filter(
          (key: IDBValidKey): key is string =>
            typeof key === "string" && key.startsWith("recording_")
        )
        .map(async (key: any) => {
          const recording = await get(key);
          if (recording && recording.blob) {
            // Create a new object URL for each recording
            return {
              ...recording,
              url: URL.createObjectURL(recording.blob),
              // Ensure duration exists, default to 0 if not present (for backward compatibility)
              duration: recording.duration || 0,
            };
          }
          return null;
        });

      recordings = (await Promise.all(recordingPromises))
        .filter(Boolean)
        .sort(
          (
            a: { date: string | number | Date },
            b: { date: string | number | Date }
          ) => new Date(b.date).getTime() - new Date(a.date).getTime()
        );
    } catch (error) {
      console.error("Error loading recordings:", error);
    }
  }

  // Start recording
  async function startRecording() {
    try {
      const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
      recorder = new MediaRecorder(stream);

      const chunks: BlobPart[] = [];
      recordingStartTime = Date.now(); // Store the start time of recording

      recorder.addEventListener("dataavailable", (e) => {
        if (e.data.size > 0) {
          chunks.push(e.data);
        }
      });

      recorder.addEventListener("stop", async () => {
        const blob = new Blob(chunks, { type: "audio/wav" });
        recordedBlob = blob;

        // Calculate the duration of the recording
        const recordingDuration = recordingStartTime
          ? (Date.now() - recordingStartTime) / 1000
          : 0;

        // Save to IndexedDB
        const id = `recording_${Date.now()}`;
        const recording: Recording = {
          id,
          blob,
          date: new Date(),
          url: URL.createObjectURL(blob),
          name: `Recording ${recordings.length + 1}`,
          duration: recordingDuration, // Store the duration in seconds
        };

        await set(id, recording);
        recordings = [recording, ...recordings];

        // Set as current recording
        setCurrentRecording(recording);
        isRecording = false;
        recordingStartTime = null; // Reset the recording start time
      });

      recorder.start();
      isRecording = true;
    } catch (error) {
      console.error("Error starting recording:", error);
    }
  }

  // Stop recording
  function stopRecording() {
    if (recorder && recorder.state !== "inactive") {
      recorder.stop();
      recorder.stream.getTracks().forEach((track) => track.stop());
    }
  }

  // Set current recording
  // 1. In the setCurrentRecording function:
  function setCurrentRecording(recording: Recording) {
    currentRecording = recording;
    if (audio) {
      audio.src = recording.url;
      audio.playbackRate = playbackRate;

      // Use the stored duration immediately
      duration = recording.duration;

      // Load the audio metadata immediately to ensure duration is available
      audio.load();

      // Add a promise to handle when audio is actually ready
      audioLoadPromise = new Promise((resolve) => {
        const loadHandler = () => {
          if (audio.duration && audio.duration !== Infinity) {
            duration = audio.duration;
            // Update stored duration if needed
            if (Math.abs(recording.duration - audio.duration) > 1) {
              updateRecordingDuration(recording, audio.duration);
            }
          }
          audio.removeEventListener("loadedmetadata", loadHandler);
          resolve();
        };
        audio.addEventListener("loadedmetadata", loadHandler);
      });
    }
  }

  let audioLoadPromise: Promise<void> | null = null;

  // Handler for when audio metadata is loaded
  function handleLoadedMetadata() {
    if (audio && currentRecording) {
      if (audio.duration && audio.duration !== Infinity) {
        duration = audio.duration;

        // If the stored duration is not accurate or missing, update it
        if (Math.abs(currentRecording.duration - audio.duration) > 1) {
          updateRecordingDuration(currentRecording, audio.duration);
        }
      } else if (currentRecording.duration > 0) {
        // Fallback to using the stored duration if the audio.duration is not available
        duration = currentRecording.duration;
      }
    }
  }

  // Update recording duration in storage
  async function updateRecordingDuration(
    recording: Recording,
    newDuration: number
  ) {
    try {
      const updatedRecording = { ...recording, duration: newDuration };
      await set(recording.id, updatedRecording);

      recordings = recordings.map((r) =>
        r.id === recording.id ? updatedRecording : r
      );

      if (currentRecording && currentRecording.id === recording.id) {
        currentRecording = updatedRecording;
      }
    } catch (error) {
      console.error("Error updating recording duration:", error);
    }
  }

  // Delete a recording
  async function deleteRecording(recording: Recording) {
    try {
      await del(recording.id);
      URL.revokeObjectURL(recording.url);

      recordings = recordings.filter((r) => r.id !== recording.id);

      if (currentRecording && currentRecording.id === recording.id) {
        currentRecording = recordings.length > 0 ? recordings[0] : null;
        if (currentRecording) {
          audio.src = currentRecording.url;
        } else {
          audio.src = "";
        }
      }
    } catch (error) {
      console.error("Error deleting recording:", error);
    }
  }

  // Rename a recording
  async function renameRecording(recording: Recording, newName: string) {
    try {
      const updatedRecording = { ...recording, name: newName };
      await set(recording.id, updatedRecording);

      recordings = recordings.map((r) =>
        r.id === recording.id ? updatedRecording : r
      );

      if (currentRecording && currentRecording.id === recording.id) {
        currentRecording = updatedRecording;
      }
    } catch (error) {
      console.error("Error renaming recording:", error);
    }
  }

  // Download recording
  function downloadRecording(recording: Recording) {
    try {
      const a = document.createElement("a");
      a.href = recording.url;
      a.download = `${recording.name}.wav`;
      document.body.appendChild(a);
      a.click();
      document.body.removeChild(a);
    } catch (error) {
      console.error("Error downloading recording:", error);
    }
  }

  // Audio event handlers
  function handlePlay() {
    isPlaying = true;
  }

  function handlePause() {
    setTimeout(() => {
      if (audio.paused) {
        isPlaying = false;
      }
    }, 75);
  }

  function handleTimeUpdate() {
    currentTime = audio.currentTime;
    if (audio.duration && audio.duration !== Infinity) {
      duration = audio.duration;

      // Update the stored duration if it's significantly different
      if (
        currentRecording &&
        Math.abs(currentRecording.duration - duration) > 1
      ) {
        updateRecordingDuration(currentRecording, duration);
      }
    }
  }

  function handleEnded() {
    if (repeat) {
      audio.currentTime = 0;
      audio.play();
    }
  }

  // Toggle play/pause
  async function togglePlay() {
    if (!currentRecording) return;

    // Ensure metadata is loaded before playing for the first time
    if (audioLoadPromise && audio.paused && duration === 0) {
      try {
        await audioLoadPromise;
      } catch (error) {
        console.error("Error loading audio metadata:", error);
      }
    }

    if (audio.paused) {
      audio.play();
    } else {
      audio.pause();
    }
  }

  // Toggle repeat
  function toggleRepeat() {
    repeat = !repeat;
  }

  // Update current time based on seeker click
  function updateCurrentTime(e: MouseEvent) {
    if (!currentRecording) return;

    const rect = seekerElement.getBoundingClientRect();
    const offsetX = e.clientX - rect.left;
    const percentage = offsetX / rect.width;

    // Use either actual audio duration or stored duration
    const effectiveDuration =
      audio.duration && audio.duration !== Infinity
        ? audio.duration
        : currentRecording.duration;

    if (effectiveDuration > 0) {
      audio.currentTime = percentage * effectiveDuration;
    }
  }

  // Change playback rate
  function changePlaybackRate(increment: number) {
    playbackRate = Math.max(Math.min(playbackRate + increment, 4), 0.4);
    if (audio) {
      audio.playbackRate = playbackRate;
    }
  }

  // Skip forward/backward
  function skip(seconds: number) {
    if (!currentRecording) return;
    audio.currentTime += seconds;
  }

  // Format time for display
  function formatTime(seconds: number): string {
    if (isNaN(seconds) || !isFinite(seconds)) return "00:00";
    const date = new Date(seconds * 1000);
    return date.toISOString().substr(14, 5);
  }

  // Get Arabic numerals
  function toArabicNumerals(s: string | number): string {
    return String(s).replace(/[0-9]/g, (d) => "٠١٢٣٤٥٦٧٨٩"[parseInt(d, 10)]);
  }
</script>

<main class="min-h-screen bg-gray-100 text-right p-4 md:p-6 font-kitab">
  <h1 class="text-3xl font-bold mb-6 text-center text-black">مسجل صوت</h1>

  <!-- Record buttons -->
  <div class="flex justify-center mb-8 gap-6">
    <!-- svelte-ignore a11y_consider_explicit_label -->
    <button
      on:click={startRecording}
      disabled={isRecording}
      class="w-16 h-16 rounded-full bg-white shadow-lg flex items-center justify-center focus:outline-none hover:bg-gray-50 disabled:opacity-50 disabled:cursor-not-allowed"
      title="سجل"
    >
      <svg viewBox="0 0 100 100" class="w-10 h-10">
        <circle
          cx="50"
          cy="50"
          r="46"
          fill={isRecording ? "#f87171" : "#ef4444"}
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

  <!-- Audio player -->
  <div
    class="bg-white shadow-lg rounded-lg p-4 mb-8 {!currentRecording
      ? 'opacity-50 pointer-events-none'
      : ''}"
  >
    <!-- Current recording name -->
    {#if currentRecording}
      <div class="flex justify-between items-center mb-2">
        <button
          on:click={() => downloadRecording(currentRecording as Recording)}
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
        <!-- Spacer for alignment -->
      </div>
    {/if}

    <!-- Seeker -->
    <!-- svelte-ignore a11y_no_static_element_interactions -->
    <!-- svelte-ignore a11y_click_events_have_key_events -->
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
              <rect x="14" y="4" width="4" height="16" /></svg
            >
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
              <polygon points="5 3 19 12 5 21 5 3" /></svg
            >
          {/if}
        </button>

        <!-- svelte-ignore a11y_consider_explicit_label -->
        <button
          on:click={toggleRepeat}
          class="w-16 h-16 rounded-full {repeat
            ? 'bg-blue-100 text-blue-600'
            : 'bg-gray-100 text-gray-600'} flex items-center justify-center focus:outline-none hover:bg-blue-100"
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
            <path d="M20 12v3a3 3 0 0 1 -3 3h-13m3 3l-3-3l3-3" /></svg
          >
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

        <div
          class="px-2 py-1 text-black text-sm bg-gray-100 rounded-md min-w-16 text-center"
        >
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

  <!-- Recordings list -->
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
                    >{new Date(recording.date).toLocaleDateString(
                      "ar-SA"
                    )}</span
                  >
                  <span class="ml-2">{formatTime(recording.duration)}</span>
                  <!-- Display saved duration -->
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

  <footer class="text-center text-gray-600 text-sm">
    من مشاريع <a
      href="https://nuqayah.com"
      target="_blank"
      class="text-blue-600 hover:underline">نُقاية</a
    >
  </footer>
</main>

<style global>
  @font-face {
    font-family: "Kitab";
    src: url("https://fonts.nuqayah.com/kitab.woff2") format("woff2");
    font-weight: normal;
    font-style: normal;
    font-display: swap;
  }

  @font-face {
    font-family: "Kitab";
    src: url("https://fonts.nuqayah.com/kitab-b.woff2") format("woff2");
    font-weight: bold;
    font-style: normal;
    font-display: swap;
  }

  :root {
    direction: rtl;
  }

  html,
  body {
    font-family: "Kitab", sans-serif;
  }

  .font-kitab {
    font-family: "Kitab", sans-serif;
  }
</style>
