# Recall.ai Desktop SDK API Usage Overview

This document provides a comprehensive guide to integrating the Recall.ai Desktop SDK for recording calls, receiving real-time transcripts, handling audio data, and managing participants.

## Table of Contents

1. [Prerequisites & Dependencies](#prerequisites--dependencies)
2. [Architecture Overview](#architecture-overview)
3. [Server Setup (Upload Token Generation)](#server-setup-upload-token-generation)
4. [SDK Initialization](#sdk-initialization)
5. [Meeting Detection & Lifecycle](#meeting-detection--lifecycle)
6. [Recording Management](#recording-management)
7. [Real-time Event Handling](#real-time-event-handling)
8. [Audio Data Processing & Storage](#audio-data-processing--storage)
9. [Transcript Processing](#transcript-processing)
10. [Participant Tracking](#participant-tracking)
11. [Complete Working Example](#complete-working-example)

---

## Prerequisites & Dependencies

### Required NPM Packages

```json
{
  "dependencies": {
    "@recallai/desktop-sdk": "^2.0.0",
    "axios": "^1.9.0",
    "express": "^4.18.2",
    "dotenv": "^16.5.0"
  }
}
```

### Environment Variables

Create a `.env` file:

```bash
RECALLAI_API_KEY=your_recall_ai_api_key_here
RECALLAI_API_URL=https://api.recall.ai
```

---

## Architecture Overview

The integration consists of three main components:

1. **Local Express Server** (port 13373) - Creates upload tokens via Recall.ai API
2. **Main Electron Process** - Handles SDK initialization, event listeners, and data management
3. **Audio Buffer System** - Stores and processes real-time audio chunks into WAV files

---

## Server Setup (Upload Token Generation)

The server creates upload tokens that enable the SDK to upload recordings to Recall.ai's servers.

### server.js - Complete Implementation

```javascript
const express = require("express");
const axios = require("axios");
const app = express();

require("dotenv").config();

const RECALLAI_API_URL = process.env.RECALLAI_API_URL || "https://api.recall.ai";
const RECALLAI_API_KEY = process.env.RECALLAI_API_KEY;

app.get("/start-recording", async (req, res) => {
  console.log(`Creating upload token with API key: ${RECALLAI_API_KEY.slice(0, 4)}...`);

  if (!RECALLAI_API_KEY) {
    console.error("RECALLAI_API_KEY is missing! Set it in .env file");
    return res.json({
      status: "error",
      message: "RECALLAI_API_KEY is missing",
    });
  }

  const url = `${RECALLAI_API_URL}/api/v1/sdk_upload/`;

  try {
    const response = await axios.post(
      url,
      {
        recording_config: {
          transcript: {
            provider: {
              // Use AssemblyAI v3 streaming for real-time transcription
              assembly_ai_v3_streaming: {},
            },
          },
          realtime_endpoints: [
            {
              type: "desktop_sdk_callback",
              events: [
                "participant_events.join",      // Participant join events
                "video_separate_png.data",      // Video frames (optional)
                "transcript.data",              // Transcript chunks
                "transcript.provider_data",     // Provider-specific transcript data
                "audio_mixed_raw.data",         // Raw audio data (16-bit PCM)
              ],
            },
          ],
        },
      },
      {
        headers: { Authorization: `Token ${RECALLAI_API_KEY}` },
        timeout: 9000,
      }
    );

    res.json({ status: "success", upload_token: response.data.upload_token });
  } catch (e) {
    console.error("Error creating upload token:", e.response?.data || e.message);
    res.json({ status: "error", message: e.message });
  }
});

// Start server on port 13373
app.listen(13373, () => {
  console.log(`Upload token server listening on http://localhost:13373`);
});

module.exports = app;
```

### Key Points:

- **Upload Token**: Required for SDK to upload recordings to Recall.ai
- **Real-time Events**: Configured in `realtime_endpoints` to receive callbacks during recording
- **Transcript Provider**: `assembly_ai_v3_streaming` provides real-time transcription
- **Port 13373**: The SDK expects this specific port for the local server

---

## SDK Initialization

### Basic SDK Setup

```javascript
const RecallAiSdk = require("@recallai/desktop-sdk");
const path = require("path");
const fs = require("fs");

// Define recording storage path
const RECORDING_PATH = path.join(app.getPath("userData"), "recordings");

// Ensure recording directory exists
if (!fs.existsSync(RECORDING_PATH)) {
  fs.mkdirSync(RECORDING_PATH, { recursive: true });
}

// Initialize the SDK
RecallAiSdk.init({
  // dev: true,  // Uncomment for development mode
  api_url: process.env.RECALLAI_API_URL,
  config: {
    audio: {
      // Enable audio capture
      enabled: true,
    },
  },
});

console.log("RecallAI SDK initialized");
```

### Important Notes:

- The SDK automatically detects meetings from supported platforms (Zoom, Google Meet, Teams, etc.)
- Audio is captured at 16kHz, mono, 16-bit PCM format
- The SDK handles screen capture and audio routing automatically

---

## Meeting Detection & Lifecycle

The SDK provides several events to track meeting state:

### 1. Meeting Detected Event

Fires when a meeting window is detected:

```javascript
RecallAiSdk.addEventListener("meeting-detected", (evt) => {
  console.log("Meeting detected!");
  console.log("Platform:", evt.window.platform);  // 'zoom', 'google-meet', 'teams', etc.
  console.log("Window ID:", evt.window.id);       // Unique identifier for this meeting
  console.log("Title:", evt.window.title);        // May be null initially
  console.log("URL:", evt.window.url);            // May be null initially
  
  // Set global flag to enable "Record Meeting" button
  global.meetingDetected = true;
  global.detectedMeeting = evt;
});
```

### 2. Meeting Updated Event

Fires when meeting metadata becomes available:

```javascript
RecallAiSdk.addEventListener("meeting-updated", async (evt) => {
  console.log("Meeting updated:", evt);
  
  const { window } = evt;
  console.log("Meeting title:", window.title);    // Now populated
  console.log("Meeting URL:", window.url);        // Now populated
  
  // Update stored meeting data with title/URL
  // This is where you'd update your database/storage
});
```

### 3. Meeting Closed Event

Fires when the meeting window closes:

```javascript
RecallAiSdk.addEventListener("meeting-closed", (evt) => {
  console.log("Meeting closed");
  console.log("Window ID:", evt.window.id);
  
  // Reset detection state
  global.meetingDetected = false;
  global.detectedMeeting = null;
});
```

---

## Recording Management

### Starting a Recording

There are two scenarios for starting recordings:

#### A. Recording a Detected Meeting (Zoom/Meet/Teams)

```javascript
async function startRecordingDetectedMeeting(meetingId) {
  try {
    // Get upload token from local server
    const uploadData = await createDesktopSdkUpload();
    
    if (!uploadData || !uploadData.upload_token) {
      console.error("Failed to get upload token");
      // Fallback: start without token
      RecallAiSdk.startRecording({
        windowId: global.detectedMeeting.window.id,
      });
      return;
    }
    
    // Store mapping of SDK window ID -> your meeting ID
    global.activeMeetingIds = global.activeMeetingIds || {};
    global.activeMeetingIds[global.detectedMeeting.window.id] = {
      platformName: global.detectedMeeting.window.platform,
      noteId: meetingId,
    };
    
    // Start recording with upload token
    RecallAiSdk.startRecording({
      windowId: global.detectedMeeting.window.id,
      uploadToken: uploadData.upload_token,
    });
    
    console.log("Recording started for:", meetingId);
  } catch (error) {
    console.error("Error starting recording:", error);
  }
}

// Helper function to get upload token
async function createDesktopSdkUpload() {
  try {
    const response = await axios.get("http://localhost:13373/start-recording", {
      timeout: 10000,
    });
    
    if (response.data.status !== "success") {
      console.error("Failed to create upload token:", response.data.message);
      return null;
    }
    
    return response.data;
  } catch (error) {
    console.error("Error creating upload token:", error);
    return null;
  }
}
```

#### B. Recording Desktop Audio (Manual/In-person)

For recording without a detected meeting (e.g., in-person meetings):

```javascript
async function startManualDesktopRecording(meetingId) {
  try {
    // Prepare desktop audio recording - this returns a unique key
    const key = await RecallAiSdk.prepareDesktopAudioRecording();
    console.log("Prepared desktop audio recording with key:", key);
    
    // Get upload token
    const uploadData = await createDesktopSdkUpload();
    if (!uploadData || !uploadData.upload_token) {
      throw new Error("Failed to create recording token");
    }
    
    // Store mapping
    global.activeMeetingIds = global.activeMeetingIds || {};
    global.activeMeetingIds[key] = {
      platformName: "Desktop Recording",
      noteId: meetingId,
    };
    
    // Start recording
    RecallAiSdk.startRecording({
      windowId: key,
      uploadToken: uploadData.upload_token,
    });
    
    return { success: true, recordingId: key };
  } catch (error) {
    console.error("Error starting manual recording:", error);
    return { success: false, error: error.message };
  }
}
```

### Stopping a Recording

```javascript
function stopRecording(recordingId) {
  try {
    console.log("Stopping recording:", recordingId);
    
    RecallAiSdk.stopRecording({
      windowId: recordingId,
    });
    
    // The 'recording-ended' event will fire automatically
    return { success: true };
  } catch (error) {
    console.error("Error stopping recording:", error);
    return { success: false, error: error.message };
  }
}
```

### Recording Ended Event

This is the most important event - it fires when recording stops:

```javascript
RecallAiSdk.addEventListener("recording-ended", async (evt) => {
  console.log("Recording ended:", evt);
  
  try {
    // 1. Save accumulated audio chunks to WAV file
    const audioFilePath = await saveAudioToWAV(evt.window.id);
    
    if (audioFilePath) {
      console.log("Audio saved to:", audioFilePath);
    }
    
    // 2. Update your database/storage with recording info
    await updateNoteWithRecordingInfo(evt.window.id);
    
    // 3. Upload to Recall.ai (optional, for cloud processing)
    setTimeout(async () => {
      const uploadData = await createDesktopSdkUpload();
      
      if (uploadData && uploadData.upload_token) {
        RecallAiSdk.uploadRecording({
          windowId: evt.window.id,
          uploadToken: uploadData.upload_token,
        });
      }
    }, 3000);
    
  } catch (error) {
    console.error("Error handling recording end:", error);
  }
});
```

### Upload Progress Tracking

```javascript
RecallAiSdk.addEventListener("upload-progress", async (evt) => {
  const { progress, window } = evt;
  console.log(`Upload progress: ${progress}%`);
  
  // Update UI with progress
  if (progress === 100) {
    console.log("Upload completed for recording:", window.id);
  }
});
```

---

## Real-time Event Handling

The SDK provides a unified `realtime-event` listener for all real-time data:

### Main Event Router

```javascript
RecallAiSdk.addEventListener("realtime-event", async (evt) => {
  // Route to appropriate handler based on event type
  
  if (evt.event === "transcript.data" && evt.data && evt.data.data) {
    await processTranscriptData(evt);
  } 
  else if (evt.event === "transcript.provider_data" && evt.data && evt.data.data) {
    await processTranscriptProviderData(evt);
  } 
  else if (evt.event === "participant_events.join" && evt.data && evt.data.data) {
    await processParticipantJoin(evt);
  } 
  else if (evt.event === "video_separate_png.data" && evt.data && evt.data.data) {
    await processVideoFrame(evt);  // Optional
  } 
  else if (evt.event === "audio_mixed_raw.data" && evt.data && evt.data.data) {
    await processAudioData(evt);
  }
});
```

---

## Audio Data Processing & Storage

### Audio Buffer Management

Create a global buffer system to accumulate audio chunks:

```javascript
const audioBuffers = {
  // Map of recordingId -> array of base64 audio chunks
  buffers: {},
  
  // Add audio chunk to a recording's buffer
  addChunk: function(recordingId, audioData) {
    if (!this.buffers[recordingId]) {
      this.buffers[recordingId] = [];
    }
    this.buffers[recordingId].push(audioData);
  },
  
  // Get all chunks for a recording
  getChunks: function(recordingId) {
    return this.buffers[recordingId] || [];
  },
  
  // Clear buffer for a recording
  clearBuffer: function(recordingId) {
    if (this.buffers[recordingId]) {
      delete this.buffers[recordingId];
      console.log(`Cleared audio buffer for recording: ${recordingId}`);
    }
  }
};
```

### Processing Audio Data Events

```javascript
async function processAudioData(evt) {
  try {
    const windowId = evt.window?.id;
    if (!windowId) {
      console.error("Missing window ID in audio data event");
      return;
    }
    
    // Check if we have this meeting in our active meetings
    if (!global.activeMeetingIds || !global.activeMeetingIds[windowId]) {
      return;  // Skip if no active meeting
    }
    
    const noteId = global.activeMeetingIds[windowId].noteId;
    if (!noteId) return;
    
    // Extract the audio data (base64 encoded, mono channel, 16K samples, S16LE)
    const audioData = evt.data.data;
    if (!audioData || !audioData.buffer) {
      return;
    }
    
    // Add the audio chunk to our buffer
    audioBuffers.addChunk(windowId, audioData.buffer);
    
    // Log occasionally to show progress (every 100 chunks)
    const chunkCount = audioBuffers.getChunks(windowId).length;
    if (chunkCount % 100 === 0) {
      console.log(`Received ${chunkCount} audio chunks for recording: ${windowId}`);
    }
  } catch (error) {
    console.error("Error processing audio data:", error);
  }
}
```

### Saving Audio to WAV File

When recording ends, convert accumulated chunks to WAV format:

```javascript
async function saveAudioToWAV(recordingId) {
  try {
    const audioChunks = audioBuffers.getChunks(recordingId);
    
    if (!audioChunks || audioChunks.length === 0) {
      console.log(`No audio chunks to save for recording: ${recordingId}`);
      return null;
    }
    
    console.log(`Saving ${audioChunks.length} audio chunks to WAV file for recording: ${recordingId}`);
    
    // Convert base64 chunks to binary buffers
    const buffers = audioChunks.map(base64Data => {
      return Buffer.from(base64Data, 'base64');
    });
    
    // Concatenate all audio data
    const totalAudioData = Buffer.concat(buffers);
    const dataSize = totalAudioData.length;
    
    // WAV file specifications (must match SDK output)
    const sampleRate = 16000;     // 16 kHz
    const numChannels = 1;        // Mono
    const bitsPerSample = 16;     // 16-bit
    const byteRate = (sampleRate * numChannels * bitsPerSample) / 8;
    const blockAlign = (numChannels * bitsPerSample) / 8;
    
    // Create WAV header (44 bytes)
    const header = Buffer.alloc(44);
    
    // RIFF chunk descriptor
    header.write('RIFF', 0);
    header.writeUInt32LE(36 + dataSize, 4);      // File size - 8
    header.write('WAVE', 8);
    
    // fmt sub-chunk
    header.write('fmt ', 12);
    header.writeUInt32LE(16, 16);                 // Subchunk1Size (16 for PCM)
    header.writeUInt16LE(1, 20);                  // AudioFormat (1 = PCM)
    header.writeUInt16LE(numChannels, 22);        // NumChannels
    header.writeUInt32LE(sampleRate, 24);         // SampleRate
    header.writeUInt32LE(byteRate, 28);           // ByteRate
    header.writeUInt16LE(blockAlign, 32);         // BlockAlign
    header.writeUInt16LE(bitsPerSample, 34);      // BitsPerSample
    
    // data sub-chunk
    header.write('data', 36);
    header.writeUInt32LE(dataSize, 40);           // Subchunk2Size
    
    // Combine header and audio data
    const wavFile = Buffer.concat([header, totalAudioData]);
    
    // Save to file using the recording ID as filename
    const audioFilePath = path.join(RECORDING_PATH, `${recordingId}.wav`);
    await fs.promises.writeFile(audioFilePath, wavFile);
    
    console.log(`Successfully saved audio file to: ${audioFilePath}`);
    console.log(`Audio file size: ${(wavFile.length / 1024 / 1024).toFixed(2)} MB`);
    console.log(`Audio duration: ~${(dataSize / byteRate).toFixed(2)} seconds`);
    
    // Clear the buffer now that we've saved the file
    audioBuffers.clearBuffer(recordingId);
    
    return audioFilePath;
  } catch (error) {
    console.error("Error saving audio to WAV file:", error);
    return null;
  }
}
```

### Audio Format Specifications

The SDK provides audio in this format:
- **Sample Rate**: 16,000 Hz (16 kHz)
- **Channels**: 1 (mono)
- **Bit Depth**: 16-bit signed little-endian (S16LE)
- **Encoding**: PCM (uncompressed)
- **Data Format**: Base64-encoded raw PCM data

---

## Transcript Processing

The SDK provides two types of transcript events:

### 1. transcript.data - General Transcript Data

```javascript
async function processTranscriptData(evt) {
  try {
    const windowId = evt.window?.id;
    if (!windowId) {
      console.error("Missing window ID in transcript event");
      return;
    }
    
    // Get the associated meeting ID
    if (!global.activeMeetingIds || !global.activeMeetingIds[windowId]) {
      return;
    }
    
    const noteId = global.activeMeetingIds[windowId].noteId;
    if (!noteId) return;
    
    // Extract transcript data
    const transcriptData = evt.data.data;
    const text = transcriptData.text || transcriptData.transcript;
    const speaker = transcriptData.speaker || "Unknown";
    
    if (!text) {
      console.log("No text in transcript data");
      return;
    }
    
    console.log(`Transcript: ${speaker}: "${text}"`);
    
    // Read your meetings data
    const meetingsData = JSON.parse(
      await fs.promises.readFile(meetingsFilePath, "utf8")
    );
    
    // Find the meeting
    const noteIndex = meetingsData.pastMeetings.findIndex(
      meeting => meeting.id === noteId
    );
    
    if (noteIndex === -1) {
      console.log("Meeting not found:", noteId);
      return;
    }
    
    const meeting = meetingsData.pastMeetings[noteIndex];
    
    // Initialize transcript array if it doesn't exist
    if (!meeting.transcript) {
      meeting.transcript = [];
    }
    
    // Add the new transcript entry
    meeting.transcript.push({
      text: text,
      speaker: speaker,
      timestamp: new Date().toISOString(),
    });
    
    console.log(`Added transcript data for meeting: ${noteId}`);
    
    // Save updated data
    await fs.promises.writeFile(
      meetingsFilePath, 
      JSON.stringify(meetingsData, null, 2)
    );
    
  } catch (error) {
    console.error("Error processing transcript data:", error);
  }
}
```

### 2. transcript.provider_data - Provider-Specific Data

This event provides additional metadata from the transcript provider (AssemblyAI):

```javascript
async function processTranscriptProviderData(evt) {
  try {
    const providerData = evt.data.data;
    
    // AssemblyAI provides additional fields:
    console.log("Provider data:", {
      text: providerData.text,
      confidence: providerData.confidence,      // Confidence score (0-1)
      words: providerData.words,                // Word-level timing
      speaker: providerData.speaker,
      is_final: providerData.is_final,          // Is this the final version?
    });
    
    // You can use this for more detailed processing
    // For example, only save transcripts marked as final:
    if (providerData.is_final) {
      // Process as shown in processTranscriptData
    }
    
  } catch (error) {
    console.error("Error processing provider transcript data:", error);
  }
}
```

### Transcript Data Structure

Each transcript entry stored should include:

```javascript
{
  text: "This is what was said",
  speaker: "John Doe",              // Speaker name
  timestamp: "2025-12-19T13:50:00Z", // ISO timestamp
  confidence: 0.95,                  // Optional: confidence score
  is_final: true                     // Optional: final vs interim
}
```

---

## Participant Tracking

Track who joins the meeting:

### Processing Participant Join Events

```javascript
async function processParticipantJoin(evt) {
  try {
    const windowId = evt.window?.id;
    if (!windowId) {
      console.error("Missing window ID in participant join event");
      return;
    }
    
    // Get the associated meeting
    if (!global.activeMeetingIds || !global.activeMeetingIds[windowId]) {
      return;
    }
    
    const noteId = global.activeMeetingIds[windowId].noteId;
    if (!noteId) return;
    
    // Extract participant data
    const participantData = evt.data.data.participant;
    if (!participantData) {
      console.log("No participant data in event");
      return;
    }
    
    const participantName = participantData.name || "Unknown Participant";
    const participantId = participantData.id;
    const isHost = participantData.is_host;
    const platform = participantData.platform;
    
    console.log(`Participant joined: ${participantName} (ID: ${participantId}, Host: ${isHost})`);
    
    // Filter out generic names
    if (
      participantName === "Host" ||
      participantName === "Guest" ||
      participantName.includes("others")
    ) {
      console.log(`Skipping generic participant name: ${participantName}`);
      return;
    }
    
    // Read and update meetings data
    const meetingsData = JSON.parse(
      await fs.promises.readFile(meetingsFilePath, "utf8")
    );
    
    const noteIndex = meetingsData.pastMeetings.findIndex(
      meeting => meeting.id === noteId
    );
    
    if (noteIndex === -1) return;
    
    const meeting = meetingsData.pastMeetings[noteIndex];
    
    // Initialize participants array if needed
    if (!meeting.participants) {
      meeting.participants = [];
    }
    
    // Check if participant already exists (based on ID)
    const existingIndex = meeting.participants.findIndex(
      p => p.id === participantId
    );
    
    const participantInfo = {
      id: participantId,
      name: participantName,
      isHost: isHost,
      platform: platform,
      joinTime: new Date().toISOString(),
      status: "active",
    };
    
    if (existingIndex !== -1) {
      // Update existing participant
      meeting.participants[existingIndex] = participantInfo;
    } else {
      // Add new participant
      meeting.participants.push(participantInfo);
    }
    
    // Save updated data
    await fs.promises.writeFile(
      meetingsFilePath,
      JSON.stringify(meetingsData, null, 2)
    );
    
    console.log(`Added participant to meeting: ${noteId}`);
    
  } catch (error) {
    console.error("Error processing participant join:", error);
  }
}
```

### Participant Data Structure

```javascript
{
  id: "participant-uuid",
  name: "John Doe",
  isHost: false,
  platform: "zoom",  // or "google-meet", "teams", etc.
  joinTime: "2025-12-19T13:45:00Z",
  status: "active"
}
```

---

## Complete Working Example

Here's a complete, minimal example that ties everything together:

```javascript
const RecallAiSdk = require("@recallai/desktop-sdk");
const axios = require("axios");
const fs = require("fs");
const path = require("path");

// Configuration
const RECORDING_PATH = "./recordings";
const DATA_FILE = "./meetings.json";

// Ensure directories exist
if (!fs.existsSync(RECORDING_PATH)) {
  fs.mkdirSync(RECORDING_PATH, { recursive: true });
}

// Audio buffer system
const audioBuffers = {
  buffers: {},
  addChunk: function(recordingId, audioData) {
    if (!this.buffers[recordingId]) {
      this.buffers[recordingId] = [];
    }
    this.buffers[recordingId].push(audioData);
  },
  getChunks: function(recordingId) {
    return this.buffers[recordingId] || [];
  },
  clearBuffer: function(recordingId) {
    delete this.buffers[recordingId];
  }
};

// Global state
global.activeMeetingIds = {};
global.meetingDetected = false;
global.detectedMeeting = null;

// Helper: Get upload token
async function createDesktopSdkUpload() {
  try {
    const response = await axios.get("http://localhost:13373/start-recording", {
      timeout: 10000,
    });
    return response.data.status === "success" ? response.data : null;
  } catch (error) {
    console.error("Error creating upload token:", error);
    return null;
  }
}

// Helper: Save audio to WAV
async function saveAudioToWAV(recordingId) {
  const audioChunks = audioBuffers.getChunks(recordingId);
  if (!audioChunks || audioChunks.length === 0) return null;
  
  const buffers = audioChunks.map(base64 => Buffer.from(base64, 'base64'));
  const totalAudioData = Buffer.concat(buffers);
  const dataSize = totalAudioData.length;
  
  // WAV specs
  const sampleRate = 16000;
  const numChannels = 1;
  const bitsPerSample = 16;
  const byteRate = (sampleRate * numChannels * bitsPerSample) / 8;
  const blockAlign = (numChannels * bitsPerSample) / 8;
  
  // Create WAV header
  const header = Buffer.alloc(44);
  header.write('RIFF', 0);
  header.writeUInt32LE(36 + dataSize, 4);
  header.write('WAVE', 8);
  header.write('fmt ', 12);
  header.writeUInt32LE(16, 16);
  header.writeUInt16LE(1, 20);
  header.writeUInt16LE(numChannels, 22);
  header.writeUInt32LE(sampleRate, 24);
  header.writeUInt32LE(byteRate, 28);
  header.writeUInt16LE(blockAlign, 32);
  header.writeUInt16LE(bitsPerSample, 34);
  header.write('data', 36);
  header.writeUInt32LE(dataSize, 40);
  
  const wavFile = Buffer.concat([header, totalAudioData]);
  const audioFilePath = path.join(RECORDING_PATH, `${recordingId}.wav`);
  
  await fs.promises.writeFile(audioFilePath, wavFile);
  audioBuffers.clearBuffer(recordingId);
  
  console.log(`Audio saved: ${audioFilePath} (${(wavFile.length / 1024 / 1024).toFixed(2)} MB)`);
  return audioFilePath;
}

// Initialize SDK
RecallAiSdk.init({
  api_url: process.env.RECALLAI_API_URL,
  config: {
    audio: { enabled: true },
  },
});

console.log("SDK initialized");

// Event Listeners

// 1. Meeting Detection
RecallAiSdk.addEventListener("meeting-detected", (evt) => {
  console.log("✅ Meeting detected:", evt.window.platform);
  global.meetingDetected = true;
  global.detectedMeeting = evt;
});

RecallAiSdk.addEventListener("meeting-closed", (evt) => {
  console.log("❌ Meeting closed");
  global.meetingDetected = false;
  global.detectedMeeting = null;
});

// 2. Recording Ended
RecallAiSdk.addEventListener("recording-ended", async (evt) => {
  console.log("🛑 Recording ended:", evt.window.id);
  
  // Save audio
  const audioFilePath = await saveAudioToWAV(evt.window.id);
  if (audioFilePath) {
    console.log("Audio file:", audioFilePath);
  }
  
  // Upload to Recall.ai (optional)
  setTimeout(async () => {
    const uploadData = await createDesktopSdkUpload();
    if (uploadData) {
      RecallAiSdk.uploadRecording({
        windowId: evt.window.id,
        uploadToken: uploadData.upload_token,
      });
    }
  }, 3000);
});

// 3. Real-time Events
RecallAiSdk.addEventListener("realtime-event", async (evt) => {
  // Audio data
  if (evt.event === "audio_mixed_raw.data" && evt.data?.data) {
    const windowId = evt.window?.id;
    const audioData = evt.data.data;
    
    if (windowId && audioData?.buffer) {
      audioBuffers.addChunk(windowId, audioData.buffer);
      
      const count = audioBuffers.getChunks(windowId).length;
      if (count % 100 === 0) {
        console.log(`📊 Received ${count} audio chunks`);
      }
    }
  }
  
  // Transcript data
  else if (evt.event === "transcript.data" && evt.data?.data) {
    const transcriptData = evt.data.data;
    const speaker = transcriptData.speaker || "Unknown";
    const text = transcriptData.text || transcriptData.transcript;
    
    if (text) {
      console.log(`💬 ${speaker}: "${text}"`);
      
      // Store transcript in your data structure
      const windowId = evt.window?.id;
      const noteId = global.activeMeetingIds[windowId]?.noteId;
      
      if (noteId) {
        // Load, update, and save your data file
        const data = JSON.parse(fs.readFileSync(DATA_FILE, "utf8"));
        const meeting = data.meetings.find(m => m.id === noteId);
        
        if (meeting) {
          if (!meeting.transcript) meeting.transcript = [];
          meeting.transcript.push({
            text: text,
            speaker: speaker,
            timestamp: new Date().toISOString(),
          });
          
          fs.writeFileSync(DATA_FILE, JSON.stringify(data, null, 2));
        }
      }
    }
  }
  
  // Participant join
  else if (evt.event === "participant_events.join" && evt.data?.data) {
    const participant = evt.data.data.participant;
    if (participant) {
      console.log(`👤 Participant joined: ${participant.name}`);
    }
  }
});

// 4. Upload Progress
RecallAiSdk.addEventListener("upload-progress", (evt) => {
  console.log(`📤 Upload progress: ${evt.progress}%`);
});

// 5. Errors
RecallAiSdk.addEventListener("error", (evt) => {
  console.error("❌ SDK Error:", evt.type, evt.message);
});

// API Functions

async function startRecording(meetingId) {
  if (!global.detectedMeeting) {
    throw new Error("No meeting detected");
  }
  
  const uploadData = await createDesktopSdkUpload();
  if (!uploadData) {
    throw new Error("Failed to get upload token");
  }
  
  const windowId = global.detectedMeeting.window.id;
  
  global.activeMeetingIds[windowId] = {
    platformName: global.detectedMeeting.window.platform,
    noteId: meetingId,
  };
  
  RecallAiSdk.startRecording({
    windowId: windowId,
    uploadToken: uploadData.upload_token,
  });
  
  console.log("▶️  Recording started");
  return windowId;
}

function stopRecording(recordingId) {
  RecallAiSdk.stopRecording({ windowId: recordingId });
  console.log("⏹️  Recording stopped");
}

// Export API
module.exports = {
  startRecording,
  stopRecording,
  isDetected: () => global.meetingDetected,
};
```

---

## Key Takeaways

### Critical Points:

1. **Server Must Run First**: The local server on port 13373 must be running before SDK initialization
2. **Upload Tokens**: Required for cloud upload; get a new token for each recording
3. **Window ID Mapping**: Map SDK window IDs to your internal meeting IDs using `global.activeMeetingIds`
4. **Audio Format**: 16kHz, mono, 16-bit PCM - must match WAV header exactly
5. **Buffer Management**: Accumulate audio chunks during recording, save on `recording-ended`
6. **Event Timing**: `meeting-detected` fires first, but `meeting-updated` has complete metadata
7. **Real-time Events**: All real-time data comes through one `realtime-event` listener

### Data Flow:

```
Meeting Window Detected
  ↓
User Starts Recording
  ↓
Get Upload Token (server)
  ↓
SDK.startRecording()
  ↓
Real-time Events Stream In:
  - audio_mixed_raw.data → Buffer accumulation
  - transcript.data → Display/store
  - participant_events.join → Track attendees
  ↓
User Stops Recording
  ↓
SDK.stopRecording()
  ↓
recording-ended Event Fires
  ↓
Save Audio Buffer → WAV File
  ↓
Upload to Recall.ai (optional)
  ↓
Done
```

### Common Pitfalls:

1. **Not waiting for upload token** - Always check token before starting recording
2. **Missing window ID mapping** - Events only provide window ID, need to map to your IDs
3. **Wrong audio format** - WAV header must match 16kHz/mono/16-bit exactly
4. **Not clearing buffers** - Memory leak if you don't clear buffers after saving
5. **Uploading too early** - Wait 2-3 seconds after recording ends before uploading

---

## Testing Checklist

- [ ] Server running on port 13373
- [ ] SDK initialized successfully
- [ ] Meeting detection working (test with Zoom/Meet)
- [ ] Recording starts with valid upload token
- [ ] Audio chunks accumulating in buffer
- [ ] Transcript events received and stored
- [ ] Participant join events tracked
- [ ] Recording stops cleanly
- [ ] Audio saved to valid WAV file
- [ ] WAV file plays correctly (test in media player)
- [ ] Upload to Recall.ai succeeds (if using)
- [ ] Memory cleaned up (buffers cleared)

---

This document should provide everything needed to build a similar application using the Recall.ai Desktop SDK!
