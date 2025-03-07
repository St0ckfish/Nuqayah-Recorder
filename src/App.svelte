<script lang="ts">
  import { onMount, onDestroy } from "svelte";
  import { get, set, del, keys } from "idb-keyval";
  import RecordButtons from "./components/RecordButtons.svelte";
  import AudioPlayer from "./components/AudioPlayer.svelte";
  import RecordingsList from "./components/RecordingsList.svelte";

  // Types
  type Recording = {
    id: string;
    blob: Blob;
    date: Date;
    url: string;
    name: string;
    duration: number;
  };

  // States
  let audio: HTMLAudioElement;
  let recorder: MediaRecorder | null = null;
  let isRecording = false;
  let isPlaying = false;
  let recordings: Recording[] = [];
  let currentRecording: Recording | null = null;
  let seekerElement: HTMLDivElement;
  let playbackRate = 1;
  let duration = 0;
  let currentTime = 0;
  let repeat = false;
  let loading = true;
  let recordingStartTime: number | null = null;

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
            return {
              ...recording,
              url: URL.createObjectURL(recording.blob),
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
      recordingStartTime = Date.now();

      recorder.addEventListener("dataavailable", (e) => {
        if (e.data.size > 0) {
          chunks.push(e.data);
        }
      });

      recorder.addEventListener("stop", async () => {
        const blob = new Blob(chunks, { type: "audio/wav" });

        const recordingDuration = recordingStartTime
          ? (Date.now() - recordingStartTime) / 1000
          : 0;

        const id = `recording_${Date.now()}`;
        const recording: Recording = {
          id,
          blob,
          date: new Date(),
          url: URL.createObjectURL(blob),
          name: `Recording ${recordings.length + 1}`,
          duration: recordingDuration,
        };

        await set(id, recording);
        recordings = [recording, ...recordings];

        setCurrentRecording(recording);
        isRecording = false;
        recordingStartTime = null;
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
  function setCurrentRecording(recording: Recording) {
    currentRecording = recording;
    if (audio) {
      audio.src = recording.url;
      audio.playbackRate = playbackRate;
      duration = recording.duration;
      audio.load();

      audioLoadPromise = new Promise((resolve) => {
        const loadHandler = () => {
          if (audio.duration && audio.duration !== Infinity) {
            duration = audio.duration;
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
        if (Math.abs(currentRecording.duration - audio.duration) > 1) {
          updateRecordingDuration(currentRecording, audio.duration);
        }
      } else if (currentRecording.duration > 0) {
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

  <!-- Record buttons component -->
  <RecordButtons {startRecording} {stopRecording} {isRecording} />

  <!-- Audio player component -->
  <AudioPlayer
    {currentRecording}
    bind:seekerElement
    {updateCurrentTime}
    {duration}
    {currentTime}
    {formatTime}
    {togglePlay}
    {isPlaying}
    {toggleRepeat}
    {repeat}
    {playbackRate}
    {changePlaybackRate}
    {skip}
    {downloadRecording}
  />

  <!-- Recordings list component -->
  <RecordingsList
    {loading}
    {recordings}
    {currentRecording}
    {setCurrentRecording}
    {renameRecording}
    {deleteRecording}
    {formatTime}
  />
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
