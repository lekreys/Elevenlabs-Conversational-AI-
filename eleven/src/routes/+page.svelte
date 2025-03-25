<script>
  import { onMount, onDestroy } from 'svelte';

  const sampleRate = 16000;

  let socket;
  let connected = false;
  let messages = [];
  let connectionError = null;
  let userAudioBase64 = ""; 
  let audioContext;

  // Variabel untuk rekaman mikrofon
  let isRecording = false;
  let micStream = null;

  // Queue untuk audio yang diterima (hanya yang diterima dari websocket akan diputar)
  let receivedAudioQueue = [];
  let isPlayingReceived = false;

  // Fungsi untuk menghasilkan ID unik bagi pesan
  function generateMessageId() {
    return Date.now().toString(36) + Math.random().toString(36).substr(2);
  }

  // Fungsi pembantu: truncate string base64 untuk tampilan log
  function truncateBase64(str, maxLength = 50) {
    return str.length <= maxLength ? str : str.substring(0, maxLength) + '...';
  }

  // Format tampilan data pesan
  function formatMessageData(data) {
    let formatted = { ...data };
    if (formatted.user_audio_chunk) {
      formatted.user_audio_chunk = truncateBase64(formatted.user_audio_chunk);
    }
    if (formatted.audio_event && formatted.audio_event.audio_base_64) {
      formatted.audio_event.audio_base_64 = truncateBase64(formatted.audio_event.audio_base_64);
    }
    return formatted;
  }

  // Fungsi untuk memutar audio PCM dari base64
  async function playPCMAudioFromBase64(base64Data, onEndedCallback = () => {}) {
    if (!audioContext) {
      try {
        audioContext = new (window.AudioContext || window.webkitAudioContext)({ sampleRate });
      } catch (error) {
        console.error("Failed to create audio context:", error);
        onEndedCallback();
        return;
      }
    }
    try {
      // Decode base64 ke binary
      const binary = atob(base64Data);
      const bytes = new Uint8Array(binary.length);
      for (let i = 0; i < binary.length; i++) {
        bytes[i] = binary.charCodeAt(i);
      }
      // Data dianggap raw PCM 16-bit
      const pcmData = new Int16Array(bytes.buffer);
      const audioBuffer = audioContext.createBuffer(1, pcmData.length, sampleRate);
      const channelData = audioBuffer.getChannelData(0);
      for (let i = 0; i < pcmData.length; i++) {
        channelData[i] = pcmData[i] / 32768.0; // Konversi ke float [-1, 1]
      }
      const source = audioContext.createBufferSource();
      source.buffer = audioBuffer;
      source.connect(audioContext.destination);
      source.onended = onEndedCallback;
      source.start();
      messages = [...messages, { type: 'system', text: 'Playing received audio chunk...', id: generateMessageId() }];
    } catch (error) {
      messages = [...messages, { type: 'error', text: `Error playing audio: ${error.message}`, id: generateMessageId() }];
      onEndedCallback();
    }
  }

  // Queue management untuk audio yang diterima
  function enqueueReceivedAudio(base64Audio) {
    receivedAudioQueue.push(base64Audio);
    if (!isPlayingReceived) playNextReceived();
  }
  function playNextReceived() {
    if (receivedAudioQueue.length === 0) {
      isPlayingReceived = false;
      return;
    }
    isPlayingReceived = true;
    const nextChunk = receivedAudioQueue.shift();
    playPCMAudioFromBase64(nextChunk, playNextReceived);
  }

  // Fungsi untuk menghubungkan ke WebSocket
  function connectWebSocket() {
    try {
      socket = new WebSocket('ws://127.0.0.1:8000/ws');

      socket.onopen = () => {
        connected = true;
        connectionError = null;
        messages = [...messages, { type: 'system', text: 'Connected to WebSocket', id: generateMessageId() }];
        if (!audioContext) {
          try {
            audioContext = new (window.AudioContext || window.webkitAudioContext)({ sampleRate });
          } catch (error) {
            console.error("Failed to initialize audio context:", error);
          }
        }
      };

      socket.onmessage = (event) => {
        try {
          const data = JSON.parse(event.data);
          const messageId = generateMessageId();

          // Jika menerima pesan ping, balas dengan pong (ikut event_id)
          if (data.type === 'ping' && data.ping_event && data.ping_event.event_id) {
            const pongMsg = { type: "pong", event_id: data.ping_event.event_id };
            sendMessage(pongMsg);
            messages = [...messages, { type: 'system', text: `Received ping, replied with pong (event_id: ${data.ping_event.event_id})`, id: generateMessageId() }];
            return;
          }

          messages = [...messages, { type: 'received', data, id: messageId }];

          // Jika pesan berisi audio (asumsi PCM 16kHz dalam base64) dari websocket
          if (data.type === 'audio' && data.audio_event && data.audio_event.audio_base_64) {
            const base64Audio = data.audio_event.audio_base_64;
            enqueueReceivedAudio(base64Audio);
          }
        } catch (error) {
          messages = [...messages, { type: 'error', text: `Failed to parse message: ${error.message}`, raw: event.data, id: generateMessageId() }];
        }
      };

      socket.onclose = (event) => {
        connected = false;
        messages = [...messages, { type: 'system', text: `Disconnected: Code ${event.code} ${event.reason}`, id: generateMessageId() }];
      };

      socket.onerror = (error) => {
        connectionError = `WebSocket error: ${error.message || 'Unknown error'}`;
        messages = [...messages, { type: 'error', text: connectionError, id: generateMessageId() }];
      };
    } catch (error) {
      connectionError = `Failed to connect: ${error.message}`;
      messages = [...messages, { type: 'error', text: connectionError, id: generateMessageId() }];
    }
  }

  // Fungsi untuk mengirim pesan via WebSocket
  function sendMessage(message) {
    if (socket && socket.readyState === WebSocket.OPEN) {
      socket.send(JSON.stringify(message));
      messages = [...messages, { type: 'sent', data: message, id: generateMessageId() }];
    } else {
      messages = [...messages, { type: 'error', text: 'Cannot send message: WebSocket not connected', id: generateMessageId() }];
    }
  }

  // Fungsi untuk memulai rekaman dari mikrofon.
  // Setiap chunk raw PCM 16-bit dikonversi ke base64 dan langsung dikirim ke WebSocket.
  async function startRecording() {
    if (isRecording) return;
    try {
      micStream = await navigator.mediaDevices.getUserMedia({
        audio: { sampleRate, channelCount: 1, echoCancellation: true, noiseSuppression: true }
      });
      if (!audioContext) {
        audioContext = new (window.AudioContext || window.webkitAudioContext)({ sampleRate });
      }
      const source = audioContext.createMediaStreamSource(micStream);
      const processor = audioContext.createScriptProcessor(4096, 1, 1);

      processor.onaudioprocess = (e) => {
        if (!isRecording) return;
        const inputData = e.inputBuffer.getChannelData(0);
        const pcmData = new Int16Array(inputData.length);
        for (let i = 0; i < inputData.length; i++) {
          const s = Math.max(-1, Math.min(1, inputData[i]));
          pcmData[i] = s < 0 ? s * 32768 : s * 32767;
        }
        // Konversi raw PCM ke base64 (tanpa header)
        const uint8Array = new Uint8Array(pcmData.buffer);
        let binary = "";
        for (let i = 0; i < uint8Array.length; i++) {
          binary += String.fromCharCode(uint8Array[i]);
        }
        const base64PCM = btoa(binary);
        // Kirim chunk ke server (data yang dikirim tidak akan diputar secara lokal)
        sendMessage({ user_audio_chunk: base64PCM });
        messages = [...messages, { type: 'system', text: 'Sent PCM audio chunk', id: generateMessageId() }];
      };

      source.connect(processor);
      processor.connect(audioContext.destination);
      isRecording = true;
      messages = [...messages, { type: 'system', text: 'Started recording from microphone (PCM 16kHz)...', id: generateMessageId() }];
    } catch (error) {
      messages = [...messages, { type: 'error', text: `Microphone error: ${error.message}`, id: generateMessageId() }];
    }
  }

  // Fungsi untuk menghentikan rekaman
  function stopRecording() {
    if (!isRecording) return;
    isRecording = false;
    if (micStream) {
      micStream.getTracks().forEach(track => track.stop());
      micStream = null;
    }
    messages = [...messages, { type: 'system', text: 'Microphone recording stopped', id: generateMessageId() }];
  }

  // Fungsi untuk mengirim test ping (saat debugging)
  function sendTestPing() {
    const testPing = {
      type: "ping",
      ping_event: { event_id: Math.floor(Math.random() * 100000), ping_ms: 50 }
    };
    sendMessage(testPing);
  }

  // Fungsi untuk mengirim dan memutar test audio (sine wave 440Hz) sebagai PCM (output)
  // Hanya untuk demonstrasi, chunk test audio yang di-*enqueue* hanya akan masuk ke receivedAudioQueue
  function playTestAudio() {
    const duration = 1; // dalam detik
    const frequency = 440;
    const numSamples = sampleRate * duration;
    const pcmData = new Int16Array(numSamples);
    for (let i = 0; i < numSamples; i++) {
      const t = i / sampleRate;
      pcmData[i] = Math.sin(2 * Math.PI * frequency * t) * 32767;
    }
    const uint8Array = new Uint8Array(pcmData.buffer);
    let binary = "";
    for (let i = 0; i < uint8Array.length; i++) {
      binary += String.fromCharCode(uint8Array[i]);
    }
    const base64 = btoa(binary);
    // Masukkan ke queue penerimaan agar diputar
    enqueueReceivedAudio(base64);
    const testAudioMessage = {
      type: "audio",
      audio_event: { audio_base_64: "base64EncodedAudioData==", event_id: Math.floor(Math.random() * 100000) }
    };
    messages = [...messages, { type: 'system', text: 'Playing test audio (440Hz tone)...', id: generateMessageId() }];
    messages = [...messages, { type: 'received', data: testAudioMessage, id: generateMessageId() }];
  }

  // Fungsi untuk meng-handle file upload audio
  async function handleFileUpload(event) {
    const file = event.target.files[0];
    if (!file) return;
    try {
      messages = [...messages, { type: 'system', text: `Processing file: ${file.name}`, id: generateMessageId() }];
      if (file.type.startsWith('audio/')) {
        convertAudioToPCM16(file);
      } else {
        const reader = new FileReader();
        reader.onload = (e) => {
          const base64 = e.target.result.split(',')[1];
          userAudioBase64 = base64;
        };
        reader.readAsDataURL(file);
      }
    } catch (error) {
      messages = [...messages, { type: 'error', text: `Error processing file: ${error.message}`, id: generateMessageId() }];
    }
  }

  // Fungsi untuk mengonversi file audio ke PCM 16kHz menggunakan OfflineAudioContext
  async function convertAudioToPCM16(file) {
    if (!audioContext) {
      try {
        audioContext = new (window.AudioContext || window.webkitAudioContext)({ sampleRate });
      } catch (error) {
        console.error("Failed to create audio context:", error);
        return;
      }
    }
    try {
      const arrayBuffer = await file.arrayBuffer();
      const audioBuffer = await audioContext.decodeAudioData(arrayBuffer);
      const offlineCtx = new OfflineAudioContext(1, audioBuffer.duration * sampleRate, sampleRate);
      const source = offlineCtx.createBufferSource();
      source.buffer = audioBuffer;
      let pcmData;
      await new Promise(resolve => {
        source.connect(offlineCtx.destination);
        source.start(0);
        offlineCtx.startRendering().then(renderedBuffer => {
          const channelData = renderedBuffer.getChannelData(0);
          pcmData = new Int16Array(channelData.length);
          for (let i = 0; i < channelData.length; i++) {
            const sample = Math.max(-1, Math.min(1, channelData[i]));
            pcmData[i] = sample < 0 ? sample * 32768 : sample * 32767;
          }
          const uint8Array = new Uint8Array(pcmData.buffer);
          let binary = "";
          for (let i = 0; i < uint8Array.length; i++) {
            binary += String.fromCharCode(uint8Array[i]);
          }
          userAudioBase64 = btoa(binary);
          resolve();
        }).catch(err => {
          console.error("Error rendering audio:", err);
          resolve();
        });
      });
      messages = [...messages, { type: 'system', text: `Audio converted to PCM 16kHz (${Math.round(pcmData.length / sampleRate * 10) / 10}s)`, id: generateMessageId() }];
    } catch (error) {
      messages = [...messages, { type: 'error', text: `Error converting audio: ${error.message}`, id: generateMessageId() }];
    }
  }

  // Fungsi untuk menutup koneksi WebSocket
  function closeConnection() {
    if (socket) {
      socket.close();
      socket = null;
    }
  }

  onMount(() => {
    connectWebSocket();
    // Resume AudioContext pada interaksi pertama (untuk mengatasi auto-play policy)
    document.addEventListener('click', function resumeAudioContext() {
      if (audioContext && audioContext.state === 'suspended') {
        audioContext.resume();
      }
      document.removeEventListener('click', resumeAudioContext);
    }, { once: true });
  });

  onDestroy(() => {
    closeConnection();
    stopRecording();
    if (audioContext) audioContext.close();
  });
</script>

<div class="w-full max-w-4xl mx-auto p-4">
  <h1 class="text-2xl font-bold mb-4">ILEPEN LEP</h1>
  
  <div class="mb-4 flex items-center">
    <div class="mr-2 h-3 w-3 rounded-full {connected ? 'bg-green-500' : 'bg-red-500'}"></div>
    <span>{connected ? 'Connected' : 'Disconnected'}</span>
    <div class="ml-auto flex">
      {#if !connected}
        <button class="bg-blue-500 hover:bg-blue-600 text-white px-4 py-1 rounded" on:click={connectWebSocket}>Connect</button>
      {:else}
        <button class="bg-red-500 hover:bg-red-600 text-white px-4 py-1 rounded" on:click={closeConnection}>Disconnect</button>
        <button class="bg-gray-500 hover:bg-gray-600 text-white px-4 py-1 rounded ml-2" on:click={sendTestPing}>Test Ping</button>
        <button class="bg-purple-500 hover:bg-purple-600 text-white px-4 py-1 rounded ml-2" on:click={playTestAudio}>Test Audio</button>
      {/if}
    </div>
  </div>
  
  {#if connectionError}
    <div class="bg-red-100 border border-red-400 text-red-700 px-4 py-3 rounded mb-4">
      {connectionError}
    </div>
  {/if}
  
  <!-- Audio Input Section -->
  <div class="mt-4 p-3 border rounded bg-gray-100">
    <h3 class="text-md font-semibold mb-2">rekam dulu boss</h3>
    <div class="flex items-center">
      <button class="bg-red-500 hover:bg-red-600 text-white px-4 py-2 rounded mr-2 {isRecording ? 'animate-pulse' : ''}" disabled={!connected || isRecording} on:click={startRecording}>
        {isRecording ? '🔴 Recording...' : '🎤 Start Recording'}
      </button>
      <button class="bg-gray-500 hover:bg-gray-600 text-white px-4 py-2 rounded" disabled={!isRecording} on:click={stopRecording}>
        Stop Recording
      </button>
    </div>
    <div class="mt-2 text-sm">
      {#if isRecording}
        <div class="text-red-600">Recording and sending each chunk as PCM 16kHz</div>
      {:else}
        <div class="text-gray-600">Click "Start Recording" to record microphone audio</div>
      {/if}
    </div>
  </div>
  
  <!-- Section untuk kirim audio manual -->

  
  <!-- Message Log -->
  <div class="border rounded-lg p-4 h-96 overflow-y-auto bg-gray-50">
    <h2 class="text-lg font-semibold mb-2">Message Log</h2>
    {#if messages.length === 0}
      <p class="text-gray-500 italic">No messages yet</p>
    {:else}
      {#each messages as message}
        <div class="mb-2 p-2 rounded {getMessageClass(message.type)}">
          <div class="text-sm font-semibold mb-1 flex justify-between">
            <span>{getMessageTitle(message.type)}</span>
            {#if message.data?.type === 'ping' || message.data?.type === 'pong'}
              <span class="text-purple-600 text-xs">
                {message.data.type.toUpperCase()}
                {#if message.data.type === 'ping'}
                  event_id: {message.data.ping_event.event_id}
                {:else}
                  event_id: {message.data.event_id}
                {/if}
              </span>
            {/if}
            {#if message.data?.type === 'audio'}
              <span class="text-green-600 text-xs">
                AUDIO event_id: {message.data.audio_event.event_id}
              </span>
            {/if}
          </div>
          {#if message.text}
            <div>{message.text}</div>
          {:else if message.data}
            {#if message.data.user_audio_chunk}
              <div class="text-blue-600">
                🎤 PCM Audio sent (input)
                <pre class="text-xs bg-blue-50 p-1 rounded">{JSON.stringify(formatMessageData(message.data), null, 2)}</pre>
              </div>
            {:else if message.data.type === 'audio' && message.data.audio_event?.audio_base_64}
              <div class="flex items-center">
                <span class="text-green-600 mr-2">🔊 PCM Audio received and played (output)</span>
                <button class="text-xs bg-green-100 hover:bg-green-200 text-green-800 px-2 py-1 rounded" on:click={() => enqueueReceivedAudio(message.data.audio_event.audio_base_64)}>
                  Play Again
                </button>
                <span class="text-xs text-gray-500 ml-2">(Base64 data truncated for display)</span>
              </div>
            {:else}
              <pre class="text-sm whitespace-pre-wrap bg-gray-100 p-2 rounded">{JSON.stringify(formatMessageData(message.data), null, 2)}</pre>
            {/if}
          {/if}
        </div>
      {/each}
    {/if}
  </div>
</div>

<style>
  pre {
    max-height: 200px;
    overflow-y: auto;
  }
</style>

<script context="module">
  function getMessageClass(type) {
    switch (type) {
      case 'sent': return 'bg-blue-100 border-l-4 border-blue-500';
      case 'received': return 'bg-green-100 border-l-4 border-green-500';
      case 'error': return 'bg-red-100 border-l-4 border-red-500';
      case 'system': return 'bg-gray-100 border-l-4 border-gray-500';
      default: return '';
    }
  }
  
  function getMessageTitle(type) {
    switch (type) {
      case 'sent': return '→ Sent';
      case 'received': return '← Received';
      case 'error': return '⚠ Error';
      case 'system': return 'ℹ System';
      default: return '';
    }
  }
</script>
