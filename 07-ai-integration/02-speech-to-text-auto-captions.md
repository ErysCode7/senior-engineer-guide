# Speech-to-Text and Auto-Captions Integration

## Overview

Speech-to-text (STT) technology converts spoken language into written text, enabling features like auto-captions, transcription services, and voice commands. This tutorial covers integration with major STT providers (Google, AWS, Azure, OpenAI Whisper), real-time transcription, multi-language support, and caption generation for video content.

## Use Cases

- **Video Content**: Automatic caption generation for accessibility
- **Meeting Transcription**: Real-time meeting notes and summaries
- **Voice Commands**: Voice-controlled applications
- **Call Centers**: Automated call transcription and analysis
- **Content Creation**: Podcast and interview transcription

## OpenAI Whisper Integration

### Setup

```bash
npm install openai form-data
```

### Basic Transcription

```typescript
// src/services/whisper.service.ts
import { OpenAI } from "openai";
import * as fs from "fs";

export class WhisperService {
  private openai: OpenAI;

  constructor() {
    this.openai = new OpenAI({
      apiKey: process.env.OPENAI_API_KEY,
    });
  }

  async transcribeAudio(
    filePath: string,
    options?: {
      language?: string;
      prompt?: string;
      responseFormat?: "json" | "text" | "srt" | "vtt" | "verbose_json";
    }
  ) {
    try {
      const transcription = await this.openai.audio.transcriptions.create({
        file: fs.createReadStream(filePath),
        model: "whisper-1",
        language: options?.language, // ISO-639-1 code (e.g., 'en', 'es')
        prompt: options?.prompt, // Optional context to improve accuracy
        response_format: options?.responseFormat || "verbose_json",
      });

      return transcription;
    } catch (error) {
      console.error("Whisper transcription error:", error);
      throw new Error("Failed to transcribe audio");
    }
  }

  async transcribeWithTimestamps(filePath: string) {
    const transcription = await this.transcribeAudio(filePath, {
      responseFormat: "verbose_json",
    });

    // verbose_json includes word-level timestamps
    return {
      text: transcription.text,
      segments: transcription.segments,
      language: transcription.language,
      duration: transcription.duration,
    };
  }

  async translateToEnglish(filePath: string) {
    // Translate any language to English
    const translation = await this.openai.audio.translations.create({
      file: fs.createReadStream(filePath),
      model: "whisper-1",
    });

    return translation.text;
  }
}

// Usage
const result = await whisperService.transcribeAudio("/path/to/audio.mp3", {
  language: "en",
  responseFormat: "verbose_json",
});
```

### Generate SRT/VTT Captions

```typescript
// src/services/caption-generator.service.ts
import { WhisperService } from "./whisper.service";
import * as fs from "fs/promises";

interface CaptionSegment {
  index: number;
  start: number;
  end: number;
  text: string;
}

export class CaptionGeneratorService {
  constructor(private readonly whisperService: WhisperService) {}

  async generateSRT(audioPath: string, outputPath: string) {
    // Get transcription with timestamps
    const transcription = await this.whisperService.transcribeWithTimestamps(
      audioPath
    );

    // Convert to SRT format
    const srtContent = transcription.segments
      .map((segment, index) => {
        const startTime = this.formatSRTTime(segment.start);
        const endTime = this.formatSRTTime(segment.end);

        return `${index + 1}
${startTime} --> ${endTime}
${segment.text.trim()}
`;
      })
      .join("\n");

    await fs.writeFile(outputPath, srtContent, "utf-8");

    return {
      path: outputPath,
      segments: transcription.segments.length,
      duration: transcription.duration,
    };
  }

  async generateVTT(audioPath: string, outputPath: string) {
    const transcription = await this.whisperService.transcribeWithTimestamps(
      audioPath
    );

    // VTT format starts with "WEBVTT"
    let vttContent = "WEBVTT\n\n";

    vttContent += transcription.segments
      .map((segment) => {
        const startTime = this.formatVTTTime(segment.start);
        const endTime = this.formatVTTTime(segment.end);

        return `${startTime} --> ${endTime}
${segment.text.trim()}
`;
      })
      .join("\n");

    await fs.writeFile(outputPath, vttContent, "utf-8");

    return {
      path: outputPath,
      segments: transcription.segments.length,
      duration: transcription.duration,
    };
  }

  private formatSRTTime(seconds: number): string {
    const hours = Math.floor(seconds / 3600);
    const minutes = Math.floor((seconds % 3600) / 60);
    const secs = Math.floor(seconds % 60);
    const ms = Math.floor((seconds % 1) * 1000);

    return `${hours.toString().padStart(2, "0")}:${minutes
      .toString()
      .padStart(2, "0")}:${secs.toString().padStart(2, "0")},${ms
      .toString()
      .padStart(3, "0")}`;
  }

  private formatVTTTime(seconds: number): string {
    const hours = Math.floor(seconds / 3600);
    const minutes = Math.floor((seconds % 3600) / 60);
    const secs = Math.floor(seconds % 60);
    const ms = Math.floor((seconds % 1) * 1000);

    return `${hours.toString().padStart(2, "0")}:${minutes
      .toString()
      .padStart(2, "0")}:${secs.toString().padStart(2, "0")}.${ms
      .toString()
      .padStart(3, "0")}`;
  }

  async generateMultiLineCaption(
    audioPath: string,
    maxCharsPerLine: number = 42,
    maxLinesPerCaption: number = 2
  ): Promise<CaptionSegment[]> {
    const transcription = await this.whisperService.transcribeWithTimestamps(
      audioPath
    );

    const captions: CaptionSegment[] = [];
    let captionIndex = 0;

    for (const segment of transcription.segments) {
      const words = segment.text.trim().split(" ");
      let currentLine = "";
      let lines: string[] = [];

      for (const word of words) {
        const testLine = currentLine ? `${currentLine} ${word}` : word;

        if (testLine.length > maxCharsPerLine) {
          if (currentLine) {
            lines.push(currentLine);
            currentLine = word;
          } else {
            lines.push(word);
            currentLine = "";
          }

          // Create caption when we hit max lines
          if (lines.length >= maxLinesPerCaption) {
            captions.push({
              index: ++captionIndex,
              start: segment.start,
              end: segment.end,
              text: lines.join("\n"),
            });
            lines = [];
          }
        } else {
          currentLine = testLine;
        }
      }

      // Add remaining text
      if (currentLine) {
        lines.push(currentLine);
      }

      if (lines.length > 0) {
        captions.push({
          index: ++captionIndex,
          start: segment.start,
          end: segment.end,
          text: lines.join("\n"),
        });
      }
    }

    return captions;
  }
}
```

## Google Cloud Speech-to-Text

### Setup

```bash
npm install @google-cloud/speech
```

### Real-Time Streaming Transcription

```typescript
// src/services/google-stt.service.ts
import speech from "@google-cloud/speech";
import { Readable } from "stream";

export class GoogleSTTService {
  private client: speech.SpeechClient;

  constructor() {
    this.client = new speech.SpeechClient({
      keyFilename: process.env.GOOGLE_APPLICATION_CREDENTIALS,
    });
  }

  async transcribeFile(
    filePath: string,
    options?: {
      languageCode?: string;
      enableAutomaticPunctuation?: boolean;
      enableWordTimeOffsets?: boolean;
    }
  ) {
    const [response] = await this.client.recognize({
      config: {
        encoding: "LINEAR16",
        sampleRateHertz: 16000,
        languageCode: options?.languageCode || "en-US",
        enableAutomaticPunctuation: options?.enableAutomaticPunctuation ?? true,
        enableWordTimeOffsets: options?.enableWordTimeOffsets ?? true,
      },
      audio: {
        uri: filePath.startsWith("gs://") ? filePath : undefined,
        content: filePath.startsWith("gs://")
          ? undefined
          : (await import("fs")).readFileSync(filePath).toString("base64"),
      },
    });

    return {
      transcription: response.results
        .map((result) => result.alternatives[0].transcript)
        .join("\n"),
      words: response.results.flatMap((result) =>
        result.alternatives[0].words.map((word) => ({
          word: word.word,
          startTime: word.startTime.seconds + word.startTime.nanos / 1e9,
          endTime: word.endTime.seconds + word.endTime.nanos / 1e9,
          confidence: result.alternatives[0].confidence,
        }))
      ),
    };
  }

  async *streamingTranscribe(
    audioStream: Readable,
    options?: {
      languageCode?: string;
      interimResults?: boolean;
    }
  ): AsyncGenerator<string> {
    const recognizeStream = this.client
      .streamingRecognize({
        config: {
          encoding: "LINEAR16",
          sampleRateHertz: 16000,
          languageCode: options?.languageCode || "en-US",
          enableAutomaticPunctuation: true,
        },
        interimResults: options?.interimResults ?? true,
      })
      .on("error", (error) => {
        console.error("Streaming error:", error);
      });

    audioStream.pipe(recognizeStream);

    for await (const data of recognizeStream) {
      if (data.results[0] && data.results[0].alternatives[0]) {
        const transcript = data.results[0].alternatives[0].transcript;
        const isFinal = data.results[0].isFinal;

        yield JSON.stringify({
          transcript,
          isFinal,
          confidence: data.results[0].alternatives[0].confidence,
        });
      }
    }
  }

  async transcribeWithSpeakerDiarization(
    filePath: string,
    minSpeakers: number = 2,
    maxSpeakers: number = 6
  ) {
    const [operation] = await this.client.longRunningRecognize({
      config: {
        encoding: "LINEAR16",
        sampleRateHertz: 16000,
        languageCode: "en-US",
        enableSpeakerDiarization: true,
        diarizationSpeakerCount: minSpeakers,
        minSpeakerCount: minSpeakers,
        maxSpeakerCount: maxSpeakers,
      },
      audio: {
        uri: filePath,
      },
    });

    const [response] = await operation.promise();

    return response.results.map((result) => ({
      transcript: result.alternatives[0].transcript,
      words: result.alternatives[0].words.map((word) => ({
        word: word.word,
        speakerTag: word.speakerTag,
        startTime: word.startTime.seconds + word.startTime.nanos / 1e9,
        endTime: word.endTime.seconds + word.endTime.nanos / 1e9,
      })),
    }));
  }
}
```

## AWS Transcribe Integration

### Setup

```bash
npm install @aws-sdk/client-transcribe @aws-sdk/client-s3
```

### Batch Transcription

```typescript
// src/services/aws-transcribe.service.ts
import {
  TranscribeClient,
  StartTranscriptionJobCommand,
  GetTranscriptionJobCommand,
  TranscriptionJob,
} from "@aws-sdk/client-transcribe";
import { S3Client, GetObjectCommand } from "@aws-sdk/client-s3";

export class AWSTranscribeService {
  private transcribeClient: TranscribeClient;
  private s3Client: S3Client;

  constructor() {
    const region = process.env.AWS_REGION || "us-east-1";

    this.transcribeClient = new TranscribeClient({ region });
    this.s3Client = new S3Client({ region });
  }

  async startTranscriptionJob(
    jobName: string,
    audioFileUri: string,
    options?: {
      languageCode?: string;
      outputBucketName?: string;
      showSpeakerLabels?: boolean;
      maxSpeakers?: number;
    }
  ) {
    const command = new StartTranscriptionJobCommand({
      TranscriptionJobName: jobName,
      LanguageCode: options?.languageCode || "en-US",
      MediaFormat: "mp3",
      Media: {
        MediaFileUri: audioFileUri,
      },
      OutputBucketName: options?.outputBucketName,
      Settings: options?.showSpeakerLabels
        ? {
            ShowSpeakerLabels: true,
            MaxSpeakerLabels: options.maxSpeakers || 2,
          }
        : undefined,
    });

    const response = await this.transcribeClient.send(command);

    return {
      jobName: response.TranscriptionJob.TranscriptionJobName,
      status: response.TranscriptionJob.TranscriptionJobStatus,
    };
  }

  async getTranscriptionJob(jobName: string): Promise<TranscriptionJob> {
    const command = new GetTranscriptionJobCommand({
      TranscriptionJobName: jobName,
    });

    const response = await this.transcribeClient.send(command);
    return response.TranscriptionJob;
  }

  async waitForTranscription(
    jobName: string,
    pollIntervalMs: number = 5000
  ): Promise<any> {
    let job = await this.getTranscriptionJob(jobName);

    while (job.TranscriptionJobStatus === "IN_PROGRESS") {
      await this.sleep(pollIntervalMs);
      job = await this.getTranscriptionJob(jobName);
    }

    if (job.TranscriptionJobStatus === "COMPLETED") {
      // Fetch transcript from S3
      const transcriptUri = job.Transcript.TranscriptFileUri;
      const transcript = await this.fetchTranscript(transcriptUri);
      return transcript;
    } else {
      throw new Error(`Transcription failed: ${job.FailureReason}`);
    }
  }

  private async fetchTranscript(uri: string): Promise<any> {
    const response = await fetch(uri);
    return response.json();
  }

  private sleep(ms: number): Promise<void> {
    return new Promise((resolve) => setTimeout(resolve, ms));
  }
}
```

## Real-Time WebSocket Transcription

### Backend Implementation

```typescript
// src/gateways/transcription.gateway.ts
import {
  WebSocketGateway,
  WebSocketServer,
  SubscribeMessage,
  OnGatewayConnection,
  OnGatewayDisconnect,
import { Server, Socket } from "socket.io";
import { WhisperService } from "../services/whisper.service";
import * as fs from "fs";
import * as path from "path";

@WebSocketGateway({ cors: true })
export class TranscriptionGateway
  implements OnGatewayConnection, OnGatewayDisconnect
{
  @WebSocketServer()
  server: Server;

  private audioBuffers: Map<string, Buffer[]> = new Map();

  constructor(private readonly whisperService: WhisperService) {}

  handleConnection(client: Socket) {
    console.log(`Client connected: ${client.id}`);
    this.audioBuffers.set(client.id, []);
  }

  handleDisconnect(client: Socket) {
    console.log(`Client disconnected: ${client.id}`);
    this.audioBuffers.delete(client.id);
  }

  @SubscribeMessage("audio-chunk")
  async handleAudioChunk(client: Socket, audioData: ArrayBuffer) {
    const buffer = Buffer.from(audioData);
    const buffers = this.audioBuffers.get(client.id) || [];
    buffers.push(buffer);
    this.audioBuffers.set(client.id, buffers);

    // Process every 5 seconds of audio (approximate)
    const totalSize = buffers.reduce((sum, buf) => sum + buf.length, 0);
    const threshold = 16000 * 2 * 5; // 16kHz, 16-bit, 5 seconds

    if (totalSize >= threshold) {
      await this.processAudioBuffer(client.id);
    }
  }

  @SubscribeMessage("finalize-transcription")
  async handleFinalize(client: Socket) {
    await this.processAudioBuffer(client.id);
    this.audioBuffers.set(client.id, []);
  }

  private async processAudioBuffer(clientId: string) {
    const buffers = this.audioBuffers.get(clientId);

    if (!buffers || buffers.length === 0) {
      return;
    }

    try {
      // Combine buffers
      const audioBuffer = Buffer.concat(buffers);

      // Save temporarily
      const tempPath = path.join("/tmp", `${clientId}-${Date.now()}.wav`);
      await fs.promises.writeFile(tempPath, audioBuffer);

      // Transcribe
      const result = await this.whisperService.transcribeAudio(tempPath, {
        responseFormat: "json",
      });

      // Send result to client
      this.server.to(clientId).emit("transcription-result", {
        text: result.text,
        timestamp: new Date().toISOString(),
      });

      // Cleanup
      await fs.promises.unlink(tempPath);

      // Clear processed buffers
      this.audioBuffers.set(clientId, []);
    } catch (error) {
      console.error("Transcription error:", error);
      this.server.to(clientId).emit("transcription-error", {
        error: "Failed to transcribe audio",
      });
    }
  }
}
```

### Frontend Implementation

```typescript
// frontend/src/services/audio-recorder.ts
import { io, Socket } from "socket.io-client";

export class AudioRecorder {
  private socket: Socket;
  private mediaRecorder: MediaRecorder;
  private audioChunks: Blob[] = [];

  constructor(private serverUrl: string) {
    this.socket = io(serverUrl);
    this.setupSocketListeners();
  }

  private setupSocketListeners() {
    this.socket.on("transcription-result", (data) => {
      console.log("Transcription:", data.text);
      this.onTranscription?.(data.text);
    });

    this.socket.on("transcription-error", (data) => {
      console.error("Error:", data.error);
      this.onError?.(data.error);
    });
  }

  async startRecording() {
    try {
      const stream = await navigator.mediaDevices.getUserMedia({ audio: true });

      this.mediaRecorder = new MediaRecorder(stream, {
        mimeType: "audio/webm",
      });

      this.mediaRecorder.ondataavailable = (event) => {
        if (event.data.size > 0) {
          this.audioChunks.push(event.data);

          // Send chunk to server
          event.data.arrayBuffer().then((buffer) => {
            this.socket.emit("audio-chunk", buffer);
          });
        }
      };

      // Collect audio in 1-second chunks
      this.mediaRecorder.start(1000);

      console.log("Recording started");
    } catch (error) {
      console.error("Failed to start recording:", error);
      throw error;
    }
  }

  stopRecording() {
    if (this.mediaRecorder && this.mediaRecorder.state !== "inactive") {
      this.mediaRecorder.stop();
      this.mediaRecorder.stream.getTracks().forEach((track) => track.stop());

      // Finalize transcription
      this.socket.emit("finalize-transcription");

      console.log("Recording stopped");
    }
  }

  disconnect() {
    this.socket.disconnect();
  }

  // Callbacks
  onTranscription?: (text: string) => void;
  onError?: (error: string) => void;
}

// Usage in React component
import React, { useState } from "react";

export const LiveTranscription: React.FC = () => {
  const [isRecording, setIsRecording] = useState(false);
  const [transcript, setTranscript] = useState("");
  const [recorder] = useState(() => new AudioRecorder("http://localhost:3000"));

  recorder.onTranscription = (text) => {
    setTranscript((prev) => prev + " " + text);
  };

  const handleStart = async () => {
    await recorder.startRecording();
    setIsRecording(true);
  };

  const handleStop = () => {
    recorder.stopRecording();
    setIsRecording(false);
  };

  return (
    <div>
      <button onClick={isRecording ? handleStop : handleStart}>
        {isRecording ? "Stop Recording" : "Start Recording"}
      </button>
      <div className="transcript">
        <h3>Live Transcript:</h3>
        <p>{transcript}</p>
      </div>
    </div>
  );
};
```

## Multi-Language Support

```typescript
// src/services/multi-language-transcription.service.ts
import { WhisperService } from "./whisper.service";

interface LanguageConfig {
  code: string;
  name: string;
  whisperCode: string;
}

export class MultiLanguageTranscriptionService {
  private supportedLanguages: LanguageConfig[] = [
    { code: "en", name: "English", whisperCode: "en" },
    { code: "es", name: "Spanish", whisperCode: "es" },
    { code: "fr", name: "French", whisperCode: "fr" },
    { code: "de", name: "German", whisperCode: "de" },
    { code: "zh", name: "Chinese", whisperCode: "zh" },
    { code: "ja", name: "Japanese", whisperCode: "ja" },
    { code: "ko", name: "Korean", whisperCode: "ko" },
  ];

  constructor(private readonly whisperService: WhisperService) {}

  async transcribeWithAutoDetect(filePath: string) {
    // Whisper can auto-detect language
    const result = await this.whisperService.transcribeAudio(filePath, {
      responseFormat: "verbose_json",
    });

    return {
      text: result.text,
      detectedLanguage: result.language,
      languageName: this.getLanguageName(result.language),
    };
  }

  async transcribeMultiLanguage(filePath: string, targetLanguages: string[]) {
    const results = await Promise.all(
      targetLanguages.map(async (lang) => {
        const result = await this.whisperService.transcribeAudio(filePath, {
          language: lang,
          responseFormat: "json",
        });

        return {
          language: lang,
          languageName: this.getLanguageName(lang),
          text: result.text,
        };
      })
    );

    return results;
  }

  private getLanguageName(code: string): string {
    return (
      this.supportedLanguages.find((l) => l.whisperCode === code)?.name || code
    );
  }

  getSupportedLanguages(): LanguageConfig[] {
    return this.supportedLanguages;
  }
}
```

## Best Practices

### 1. **Audio Quality Matters**

- Use high-quality audio (16kHz+ sample rate)
- Minimize background noise
- Use proper microphone equipment

### 2. **Optimize File Sizes**

- Compress audio before uploading
- Use appropriate formats (MP3, M4A, WAV)
- Consider streaming for long recordings

### 3. **Handle Errors Gracefully**

- Implement retry logic for transient failures
- Provide fallback options
- Log errors for debugging

### 4. **Privacy and Security**

- Encrypt audio files in transit and at rest
- Delete temporary files after processing
- Comply with data protection regulations

### 5. **Cost Management**

- Cache transcriptions when possible
- Use batch processing for non-real-time needs
- Monitor API usage and costs

### 6. **User Experience**

- Show interim results for real-time transcription
- Provide progress indicators for batch jobs
- Allow manual correction of transcripts

### 7. **Accessibility**

- Always provide captions for video content
- Support multiple languages
- Follow WCAG guidelines

## Summary

Speech-to-text integration enables powerful accessibility and productivity features. Focus on:

- Choosing the right STT provider for your needs
- Implementing real-time streaming for live transcription
- Generating properly formatted captions (SRT/VTT)
- Supporting multiple languages
- Optimizing audio quality and file sizes
- Managing costs and API usage
- Ensuring privacy and security

These practices ensure reliable, high-quality transcription services in production applications.
