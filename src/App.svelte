<script lang="ts">
  import { onDestroy, onMount } from "svelte";
  import { Peer } from "peerjs";
  import type { DataConnection } from "peerjs";

  type Message = {
    sender: "self" | "peer" | "system";
    text: string;
    file?: FileMessage;
    uploadProgress?: number;
  };

  type FileMessage = {
    name: string;
    type: string;
    size: number;
    dataUrl: string;
  };

  type FileMeta = Omit<FileMessage, "dataUrl">;

  type PeerPayload =
    | string
    | {
        kind: "file-start";
        id: string;
        file: FileMeta;
      }
    | {
        kind: "file-chunk";
        id: string;
        chunk: ArrayBuffer;
        loaded: number;
        total: number;
      }
    | {
        kind: "file-end";
        id: string;
      };

  type IncomingTransfer = {
    messageIndex: number;
    chunks: ArrayBuffer[];
    file: FileMeta;
  };

  type Theme = "light" | "dark";

  let peer = $state<Peer>();
  let peerId = $state(getId());
  let targetPeerId = $state("");
  let connection = $state<DataConnection>();
  let fileInput = $state<HTMLInputElement>();
  let messages = $state<Message[]>([]);
  let messageText = $state("");
  let theme = $state<Theme>("dark");
  let connectionState = $state<"waiting" | "connecting" | "connected">(
    "waiting",
  );
  let copied = $state(false);
  let copiedMessageIndex = $state<number | null>(null);

  const incomingTransfers = new Map<string, IncomingTransfer>();
  const fileChunkSize = 64 * 1024;

  const statusText = $derived(
    connectionState === "connected"
      ? "Connected"
      : connectionState === "connecting"
        ? "Connecting"
        : "Ready",
  );

  function getId(): string {
    if (localStorage.getItem("peerclip-peer-id")) {
      return localStorage.getItem("peerclip-peer-id")!;
    } else {
      const newId = Math.random().toString(36).substring(2, 10);
      localStorage.setItem("peerclip-peer-id", newId);
      return newId;
    }
  }

  onMount(() => {
    const savedTheme = localStorage.getItem("peerclip-theme") as Theme | null;
    const preferredTheme = window.matchMedia("(prefers-color-scheme: light)")
      .matches
      ? "light"
      : "dark";

    theme = savedTheme ?? preferredTheme;
    applyTheme(theme);
    initializePeer(peerId);
  });

  function initializePeer(id: string) {
    peer = new Peer(id);

    peer.on("connection", (conn: DataConnection) => {
      connection = conn;
      setupConnectionListeners(conn);
    });
  }

  function applyTheme(nextTheme: Theme) {
    document.documentElement.dataset.theme = nextTheme;
  }

  function toggleTheme() {
    theme = theme === "dark" ? "light" : "dark";
    localStorage.setItem("peerclip-theme", theme);
    applyTheme(theme);
  }

  async function copyPeerId() {
    if (!peerId) return;

    await navigator.clipboard.writeText(peerId);
    copied = true;
    setTimeout(() => {
      copied = false;
    }, 1600);
  }

  function resetPeerId() {
    const nextPeerId = Math.random().toString(36).substring(2, 10);

    connection?.close();
    peer?.destroy();
    localStorage.setItem("peerclip-peer-id", nextPeerId);
    peerId = nextPeerId;
    connection = undefined;
    connectionState = "waiting";
    messages.push({ sender: "system", text: "Peer ID reset." });
    initializePeer(nextPeerId);
  }

  async function copyPeerMessage(text: string, index: number) {
    await navigator.clipboard.writeText(text);
    copiedMessageIndex = index;
    setTimeout(() => {
      if (copiedMessageIndex === index) copiedMessageIndex = null;
    }, 1600);
  }

  function connectToPeer() {
    const requestedPeerId = targetPeerId.trim();

    if (!peer || !requestedPeerId) return;

    connectionState = "connecting";
    connection = peer.connect(requestedPeerId);
    setupConnectionListeners(connection);
  }

  function leavePeer() {
    if (!connection) return;

    connection.close();
    connection = undefined;
    connectionState = "waiting";
  }

  function setupConnectionListeners(conn: DataConnection) {
    conn.on("open", () => {
      connectionState = "connected";
      messages.push({ sender: "system", text: "Secure peer channel opened." });
    });

    conn.on("data", (data: unknown) => {
      handlePeerData(data);
    });

    conn.on("close", () => {
      connectionState = "waiting";
      messages.push({ sender: "system", text: "Peer disconnected." });
    });

    conn.on("error", () => {
      connectionState = "waiting";
      messages.push({
        sender: "system",
        text: "Connection failed. Check the peer ID and try again.",
      });
    });
  }

  function sendMessage() {
    const outgoing = messageText.trim();

    if (!connection || !outgoing) return;

    connection.send(outgoing);
    messages.push({ sender: "self", text: outgoing });
    messageText = "";
  }

  function clearChat() {
    messages = [];
    copiedMessageIndex = null;
  }

  function handlePeerData(data: unknown) {
    if (!isPeerPayload(data)) {
      messages.push({ sender: "peer", text: String(data) });
      return;
    }

    if (typeof data === "string") {
      messages.push({ sender: "peer", text: data });
      return;
    }

    if (data.kind === "file-start") {
      const messageIndex = messages.length;

      messages.push({
        sender: "peer",
        text: `File: ${data.file.name}`,
        file: { ...data.file, dataUrl: "" },
        uploadProgress: 0,
      });
      incomingTransfers.set(data.id, {
        messageIndex,
        chunks: [],
        file: data.file,
      });
      return;
    }

    if (data.kind === "file-chunk") {
      const transfer = incomingTransfers.get(data.id);

      if (!transfer) return;

      transfer.chunks.push(data.chunk);
      updateMessage(transfer.messageIndex, {
        uploadProgress: Math.max(
          1,
          Math.round((data.loaded / data.total) * 100),
        ),
      });
      return;
    }

    if (data.kind === "file-end") {
      const transfer = incomingTransfers.get(data.id);

      if (!transfer) return;

      const blob = new Blob(transfer.chunks, { type: transfer.file.type });
      updateMessage(transfer.messageIndex, {
        file: {
          ...transfer.file,
          dataUrl: URL.createObjectURL(blob),
        },
        uploadProgress: undefined,
      });
      incomingTransfers.delete(data.id);
    }
  }

  function updateMessage(index: number, patch: Partial<Message>) {
    if (!messages[index]) return;

    messages[index] = { ...messages[index], ...patch };
    messages = [...messages];
  }

  function isPeerPayload(data: unknown): data is PeerPayload {
    return (
      typeof data === "string" ||
      (typeof data === "object" &&
        data !== null &&
        "kind" in data &&
        (data.kind === "file-start" ||
          data.kind === "file-chunk" ||
          data.kind === "file-end"))
    );
  }

  function createTransferId() {
    return crypto.randomUUID?.() ?? Math.random().toString(36).slice(2);
  }

  function isFilePayload(
    data: unknown,
  ): data is Extract<PeerPayload, { kind: "file-start" }> {
    return (
      typeof data === "object" &&
      data !== null &&
      "kind" in data &&
      data.kind === "file-start" &&
      "file" in data
    );
  }

  function formatFileSize(size: number) {
    if (size < 1024) return `${size} B`;
    if (size < 1024 * 1024) return `${Math.round(size / 1024)} KB`;
    return `${(size / 1024 / 1024).toFixed(1)} MB`;
  }

  function chooseFile() {
    fileInput?.click();
  }

  async function sendFile(event: Event) {
    const input = event.currentTarget as HTMLInputElement;
    const file = input.files?.[0];

    if (!connection || !file) return;

    const transferId = createTransferId();
    const messageIndex = messages.length;
    const pendingFile = {
      name: file.name,
      type: file.type || "application/octet-stream",
      size: file.size,
      dataUrl: "",
    };

    messages.push({
      sender: "self",
      text: `File: ${file.name}`,
      file: pendingFile,
      uploadProgress: 0,
    });

    const buffer = await file.arrayBuffer();
    const blobUrl = URL.createObjectURL(file);
    const fileMessage = {
      name: file.name,
      type: file.type || "application/octet-stream",
      size: file.size,
      dataUrl: blobUrl,
    };

    connection.send({
      kind: "file-start",
      id: transferId,
      file: {
        name: file.name,
        type: file.type || "application/octet-stream",
        size: file.size,
      },
    } satisfies PeerPayload);

    for (let offset = 0; offset < buffer.byteLength; offset += fileChunkSize) {
      const nextOffset = Math.min(offset + fileChunkSize, buffer.byteLength);
      const chunk = buffer.slice(offset, nextOffset);

      connection.send({
        kind: "file-chunk",
        id: transferId,
        chunk,
        loaded: nextOffset,
        total: buffer.byteLength,
      } satisfies PeerPayload);
      updateMessage(messageIndex, {
        uploadProgress: Math.max(
          1,
          Math.round((nextOffset / buffer.byteLength) * 100),
        ),
      });
    }

    connection.send({
      kind: "file-end",
      id: transferId,
    } satisfies PeerPayload);
    updateMessage(messageIndex, {
      file: fileMessage,
      uploadProgress: 100,
    });
    setTimeout(() => {
      if (messages[messageIndex]?.file?.dataUrl === blobUrl) {
        updateMessage(messageIndex, { uploadProgress: undefined });
      }
    }, 600);
    input.value = "";
  }

  onDestroy(() => {
    peer?.destroy();
  });
</script>

<svelte:head>
  <title>Peerclip</title>
</svelte:head>

<main class="app-shell">
  <header class="topbar" aria-label="Peerclip controls">
    <a class="brand" href="/" aria-label="Peerclip home">
      <span class="brand-mark">P</span>
      <span>
        <strong>Peerclip</strong>
        <small>Peer to peer text and file sharing</small>
      </span>
    </a>

    <div class="topbar-actions">
      <div class="status-pill" data-state={connectionState}>
        <span aria-hidden="true"></span>
        {statusText}
      </div>

      <button
        class="icon-button"
        type="button"
        onclick={toggleTheme}
        aria-label="Toggle theme"
      >
        {theme === "dark" ? "☀" : "☾"}
      </button>
    </div>
  </header>

  <section class="workspace" aria-label="Peer chat workspace">
    <aside class="panel connect-panel" aria-label="Connection setup">
      <div>
        <p class="eyebrow">Your peer ID</p>
        <div class="peer-id-row">
          <code class="peer-id">{peerId || "Starting peer..."}</code>
          <button
            class="ghost-button peer-id-action"
            type="button"
            onclick={copyPeerId}
            disabled={!peerId}
            aria-label="Copy peer ID"
            title="Copy peer ID"
          >
            <span aria-hidden="true">{copied ? "✓" : "⧉"}</span>
          </button>
          <button
            class="ghost-button peer-id-action"
            type="button"
            onclick={resetPeerId}
            aria-label="Reset peer ID"
            title="Reset peer ID"
          >
            <span aria-hidden="true">↻</span>
          </button>
        </div>
      </div>

      <form
        class="connect-form"
        onsubmit={(event) => {
          event.preventDefault();
          connectToPeer();
        }}
      >
        <label for="target-peer">Connect to peer</label>
        <div class="input-action">
          <input
            id="target-peer"
            type="text"
            bind:value={targetPeerId}
            placeholder="Paste a peer ID"
            autocomplete="off"
          />
          <button
            class="primary-button connect-action"
            type="submit"
            disabled={!targetPeerId.trim()}
            aria-label="Connect to peer"
            title="Connect to peer"
          >
            <span aria-hidden="true">↗</span>
          </button>
          <button
            class="ghost-button connect-action"
            type="button"
            onclick={leavePeer}
            disabled={!connection}
            aria-label="Leave peer"
            title="Leave peer"
          >
            <span aria-hidden="true">×</span>
          </button>
        </div>
      </form>
    </aside>

    <section class="chat-panel" aria-label="Messages">
      <div class="copy-note">
        <span>Click a received text message to copy it.</span>
        <button
          class="ghost-button clear-chat-button"
          type="button"
          onclick={clearChat}
          disabled={messages.length === 0}
          aria-label="Clear chat"
          title="Clear chat"
        >
          <span aria-hidden="true">⌫</span>
        </button>
      </div>

      <div class="chat-window" aria-live="polite">
        {#if messages.length === 0}
          <div class="empty-state">
            <div class="empty-mark">GO</div>
            <h2>Nothing sent yet</h2>
            <p>
              Copy your ID, pass it over, then start dropping text into the
              channel.
            </p>
          </div>
        {:else}
          {#each messages as msg, index}
            {#if msg.sender === "peer" && !msg.file}
              <button
                class="message peer"
                type="button"
                onclick={() => copyPeerMessage(msg.text, index)}
                aria-label="Copy peer message"
              >
                <p>{msg.text}</p>
                {#if copiedMessageIndex === index}
                  <span class="message-hint" aria-hidden="true">Copied</span>
                {/if}
              </button>
            {:else}
              <article class={`message ${msg.sender}`}>
                {#if msg.file}
                  <svelte:element
                    this={msg.file.dataUrl ? "a" : "div"}
                    class="file-message"
                    href={msg.file.dataUrl || undefined}
                    download={msg.file.dataUrl ? msg.file.name : undefined}
                  >
                    <span
                      class:pending-file-icon={!msg.file.dataUrl}
                      aria-hidden="true"
                    >
                      {#if msg.file.dataUrl}
                        <svg viewBox="0 0 24 24" focusable="false">
                          <path d="M12 3v11" />
                          <path d="m7 10 5 5 5-5" />
                          <path d="M5 21h14" />
                        </svg>
                      {:else}
                        <svg viewBox="0 0 24 24" focusable="false">
                          <path d="M12 3a9 9 0 1 0 9 9" />
                        </svg>
                      {/if}
                    </span>
                    <span>
                      <strong>{msg.file.name}</strong>
                      <small>
                        {#if msg.uploadProgress !== undefined}
                          {msg.sender === "peer" ? "Retrieving" : "Uploading"}, {msg.uploadProgress}%
                        {:else}
                          {formatFileSize(msg.file.size)}
                        {/if}
                      </small>
                    </span>
                  </svelte:element>
                  {#if msg.uploadProgress !== undefined}
                    <div
                      class="upload-progress"
                      aria-label={`${msg.sender === "peer" ? "Retrieving" : "Uploading"} ${msg.file.name}: ${msg.uploadProgress}%`}
                    >
                      <span style={`width: ${msg.uploadProgress}%`}></span>
                    </div>
                  {/if}
                {:else}
                  <p>{msg.text}</p>
                {/if}
              </article>
            {/if}
          {/each}
        {/if}
      </div>

      <form
        class="message-form"
        onsubmit={(event) => {
          event.preventDefault();
          sendMessage();
        }}
      >
        <label class="sr-only" for="message-text">Message</label>
        <input
          id="message-text"
          type="text"
          bind:value={messageText}
          placeholder={connectionState === "connected"
            ? "Type a message"
            : "Connect before sending"}
          disabled={connectionState !== "connected"}
          autocomplete="off"
        />
        <input
          class="file-input"
          type="file"
          bind:this={fileInput}
          onchange={sendFile}
          disabled={connectionState !== "connected"}
          aria-label="Choose file to send"
        />
        <button
          class="ghost-button message-action"
          type="button"
          onclick={chooseFile}
          disabled={connectionState !== "connected"}
          aria-label="Upload file"
          title="Upload file"
        >
          <span aria-hidden="true">⇧</span>
        </button>
        <button
          class="primary-button message-action"
          type="submit"
          disabled={connectionState !== "connected" || !messageText.trim()}
          aria-label="Send message"
          title="Send message"
        >
          <span aria-hidden="true">↗</span>
        </button>
      </form>
    </section>
  </section>
</main>

<style>
  :global(:root) {
    --bg: oklch(94% 0.06 92);
    --surface: oklch(98% 0.012 92);
    --surface-raised: oklch(90% 0.05 194);
    --text: oklch(18% 0.03 255);
    --muted: oklch(34% 0.035 255);
    --soft: oklch(83% 0.07 92);
    --border: oklch(18% 0.03 255);
    --accent: oklch(70% 0.22 28);
    --accent-strong: oklch(62% 0.25 28);
    --accent-text: oklch(14% 0.028 255);
    --success: oklch(77% 0.22 145);
    --warning: oklch(83% 0.19 83);
    --chrome-text: var(--text);
    --chrome-muted: var(--muted);
    --badge-border: var(--border);
    --chat-bg: var(--surface);
    --chat-text: var(--text);
    --chat-border: var(--border);
    --chat-shadow: var(--shadow);
    --chat-shadow-small: var(--shadow-small);
    --copy-note-bg: var(--soft);
    --copy-note-text: var(--text);
    --copy-popover-bg: var(--soft);
    --copy-popover-text: var(--text);
    --copy-popover-border: var(--border);
    --peer-message-bg: var(--surface);
    --peer-message-text: var(--text);
    --self-message-bg: var(--accent);
    --self-message-text: var(--accent-text);
    --system-message-text: var(--muted);
    --page-texture: color-mix(in oklch, var(--border) 14%, transparent);
    --panel-texture: color-mix(in oklch, var(--border) 12%, transparent);
    --shadow: 8px 8px 0 var(--border);
    --shadow-small: 4px 4px 0 var(--border);
    color-scheme: light;
  }

  :global(:root[data-theme="dark"]) {
    --bg: oklch(20% 0.04 278);
    --surface: oklch(24% 0.055 278);
    --surface-raised: oklch(31% 0.07 278);
    --text: oklch(96% 0.012 94);
    --muted: oklch(78% 0.035 94);
    --soft: oklch(30% 0.075 278);
    --border: oklch(72% 0.14 318);
    --accent: oklch(82% 0.18 138);
    --accent-strong: oklch(75% 0.2 138);
    --accent-text: oklch(15% 0.035 278);
    --success: oklch(82% 0.18 138);
    --warning: oklch(86% 0.18 82);
    --chrome-text: oklch(96% 0.012 94);
    --chrome-muted: oklch(82% 0.035 94);
    --badge-border: var(--border);
    --chat-bg: oklch(18% 0.045 278);
    --chat-text: oklch(96% 0.012 94);
    --chat-border: var(--border);
    --chat-shadow: 8px 8px 0 oklch(9% 0.025 278);
    --chat-shadow-small: 4px 4px 0 oklch(9% 0.025 278);
    --copy-note-bg: oklch(25% 0.065 278);
    --copy-note-text: oklch(96% 0.012 94);
    --copy-popover-bg: var(--accent);
    --copy-popover-text: var(--accent-text);
    --copy-popover-border: oklch(96% 0.012 94);
    --peer-message-bg: oklch(26% 0.055 278);
    --peer-message-text: oklch(96% 0.012 94);
    --self-message-bg: var(--accent);
    --self-message-text: var(--accent-text);
    --system-message-text: oklch(74% 0.055 318);
    --page-texture: oklch(82% 0.08 318 / 0.28);
    --panel-texture: oklch(82% 0.08 318 / 0.18);
    --shadow: 8px 8px 0 var(--border);
    --shadow-small: 4px 4px 0 var(--border);
    color-scheme: dark;
  }

  :global(*) {
    box-sizing: border-box;
  }

  :global(body) {
    min-width: 320px;
    min-height: 100svh;
    background: linear-gradient(90deg, var(--page-texture) 2px, transparent 2px),
      linear-gradient(var(--page-texture) 2px, transparent 2px), var(--bg);
    background-size: 34px 34px;
    color: var(--text);
  }

  :global(#app) {
    width: 100%;
    max-width: none;
    min-height: 100svh;
    margin: 0;
    border: 0;
    text-align: left;
  }

  button,
  input {
    font: inherit;
  }

  button {
    border: 0;
  }

  .app-shell {
    min-height: 100svh;
    padding: 28px;
  }

  .topbar {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 16px;
    max-width: 1180px;
    margin: 0 auto 24px;
  }

  .brand {
    display: inline-flex;
    align-items: center;
    gap: 14px;
    color: var(--chrome-text);
    text-decoration: none;
  }

  .brand-mark {
    display: grid;
    width: 48px;
    height: 48px;
    place-items: center;
    border: 3px solid var(--border);
    border-radius: 0;
    background: var(--accent);
    color: var(--accent-text);
    box-shadow: var(--shadow-small);
    font-size: 1.35rem;
    font-weight: 950;
  }

  .brand strong,
  .brand small {
    display: block;
  }

  .brand strong {
    line-height: 1.1;
    font-size: 1.2rem;
    font-weight: 950;
    text-transform: uppercase;
  }

  .brand small {
    color: var(--chrome-muted);
    font-size: 0.78rem;
    font-weight: 850;
    margin-top: 2px;
    text-transform: uppercase;
  }

  .topbar-actions {
    display: flex;
    align-items: center;
    gap: 10px;
  }

  .status-pill {
    display: inline-flex;
    align-items: center;
    gap: 8px;
    min-height: 40px;
    padding: 0 14px;
    border: 3px solid var(--border);
    border-radius: 0;
    background: var(--warning);
    color: var(--text);
    box-shadow: var(--shadow-small);
    font-size: 0.86rem;
    font-weight: 950;
    text-transform: uppercase;
  }

  .status-pill span {
    width: 8px;
    height: 8px;
    border: 2px solid var(--border);
    border-radius: 0;
    background: var(--warning);
  }

  .status-pill[data-state="connected"] {
    background: var(--success);
  }

  .status-pill[data-state="connecting"] {
    background: var(--surface-raised);
  }

  .status-pill[data-state="connected"] span {
    background: var(--success);
  }

  .status-pill[data-state="connecting"] span {
    background: var(--accent);
  }

  .icon-button,
  .ghost-button,
  .primary-button {
    min-height: 40px;
    border-radius: 0;
    font-weight: 950;
    cursor: pointer;
    transition:
      transform 120ms cubic-bezier(0.16, 1, 0.3, 1),
      box-shadow 120ms cubic-bezier(0.16, 1, 0.3, 1),
      background-color 180ms cubic-bezier(0.16, 1, 0.3, 1),
      border-color 180ms cubic-bezier(0.16, 1, 0.3, 1);
    text-transform: uppercase;
  }

  .icon-button:hover,
  .ghost-button:hover,
  .primary-button:hover {
    box-shadow: 2px 2px 0 var(--border);
    transform: translate(2px, 2px);
  }

  .icon-button:focus-visible,
  .ghost-button:focus-visible,
  .primary-button:focus-visible,
  input:focus-visible {
    outline: 4px solid var(--accent);
    outline-offset: 3px;
  }

  .icon-button {
    width: 42px;
    background: var(--surface-raised);
    color: var(--text);
    border: 3px solid var(--border);
    box-shadow: var(--shadow-small);
    font-size: 1rem;
  }

  .workspace {
    display: grid;
    grid-template-columns: minmax(280px, 360px) minmax(0, 1fr);
    gap: 18px;
    max-width: 1180px;
    margin: 0 auto;
  }

  .panel,
  .chat-panel {
    border: 4px solid var(--badge-border);
    border-radius: 0;
    background: var(--surface);
    box-shadow: var(--shadow);
  }

  .connect-panel {
    display: flex;
    flex-direction: column;
    gap: 26px;
    align-self: start;
    padding: 22px;
  }

  .eyebrow,
  label {
    margin: 0 0 8px;
    color: var(--text);
    font-size: 0.78rem;
    font-weight: 950;
    letter-spacing: 0.08em;
    text-transform: uppercase;
  }

  .peer-id-row,
  .input-action,
  .message-form {
    display: flex;
    gap: 10px;
  }

  .peer-id-row {
    align-items: stretch;
  }

  .peer-id {
    flex: 1;
    min-width: 0;
    padding: 13px 14px;
    border: 3px solid var(--border);
    border-radius: 0;
    background: var(--surface-raised);
    color: var(--text);
    font-family: ui-monospace, SFMono-Regular, Menlo, Consolas, monospace;
    font-size: 0.85rem;
    overflow: hidden;
    text-overflow: ellipsis;
    user-select: all;
    white-space: nowrap;
  }

  .ghost-button {
    padding: 0 14px;
    background: var(--surface);
    color: var(--text);
    border: 3px solid var(--border);
    box-shadow: var(--shadow-small);
  }

  .peer-id-action {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    width: 44px;
    min-width: 44px;
    padding-inline: 10px;
  }

  .peer-id-action span {
    font-size: 1.12rem;
    line-height: 1;
  }

  .connect-action {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    width: 48px;
    min-width: 48px;
    padding-inline: 0;
  }

  .connect-action span {
    font-size: 1.18rem;
    line-height: 1;
  }

  .primary-button {
    padding: 0 18px;
    background: var(--accent);
    color: var(--accent-text);
    border: 3px solid var(--border);
    box-shadow: var(--shadow-small);
  }

  .primary-button:hover {
    background: var(--accent-strong);
  }

  button:disabled,
  input:disabled {
    cursor: not-allowed;
    opacity: 0.55;
    transform: none;
  }

  .connect-form {
    display: grid;
    gap: 8px;
  }

  input {
    width: 100%;
    min-width: 0;
    min-height: 44px;
    border: 3px solid var(--border);
    border-radius: 0;
    background: var(--surface-raised);
    color: var(--text);
    padding: 0 14px;
    font-weight: 750;
  }

  input::placeholder {
    color: var(--muted);
  }

  .chat-panel {
    display: grid;
    grid-template-rows: auto minmax(430px, 64svh) auto;
    border-color: var(--chat-border);
    box-shadow: var(--chat-shadow);
    overflow: hidden;
  }

  .copy-note {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 12px;
    margin: 0;
    padding: 8px 10px 8px 18px;
    border-bottom: 4px solid var(--chat-border);
    background: var(--copy-note-bg);
    color: var(--copy-note-text);
    font-size: 0.82rem;
    font-weight: 950;
    text-transform: uppercase;
  }

  .copy-note > span:first-child {
    min-width: 0;
  }

  .clear-chat-button {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    width: 38px;
    min-width: 38px;
    min-height: 34px;
    padding: 0;
  }

  .clear-chat-button span {
    font-size: 1rem;
    line-height: 1;
  }

  .chat-window {
    display: flex;
    flex-direction: column;
    gap: 12px;
    overflow-y: auto;
    padding: 24px 28px;
    background:
      linear-gradient(135deg, var(--panel-texture) 25%, transparent 25%) 0 0 /
        24px 24px,
      var(--chat-bg);
  }

  .empty-state {
    display: grid;
    place-items: center;
    align-content: center;
    min-height: 100%;
    max-width: 34rem;
    margin: auto;
    text-align: center;
    color: var(--chat-text);
  }

  .empty-mark {
    display: grid;
    width: 64px;
    height: 52px;
    margin-bottom: 18px;
    place-items: center;
    border: 4px solid var(--badge-border);
    border-radius: 0;
    background: var(--accent);
    color: var(--accent-text);
    box-shadow: var(--shadow-small);
    font-size: 1rem;
    font-weight: 950;
  }

  .empty-state h2 {
    margin: 0 0 8px;
    color: var(--chat-text);
    font-size: 1.45rem;
    font-weight: 950;
    text-transform: uppercase;
  }

  .empty-state p {
    margin: 0;
    line-height: 1.55;
    font-weight: 750;
  }

  .message {
    position: relative;
    width: fit-content;
    max-width: min(70%, 46rem);
    padding: 12px 14px;
    border: 3px solid var(--chat-border);
    border-radius: 0;
    background: var(--peer-message-bg);
    box-shadow: var(--chat-shadow-small);
  }

  .message.peer {
    margin-top: 14px;
    color: var(--peer-message-text);
    cursor: copy;
    text-align: left;
  }

  .message.peer:focus-visible {
    outline: 4px solid var(--accent);
    outline-offset: 4px;
  }

  .message.self {
    align-self: flex-end;
    background: var(--self-message-bg);
    color: var(--self-message-text);
    border-color: var(--chat-border);
  }

  .message.system {
    align-self: center;
    max-width: 90%;
    padding: 4px 0;
    border: 0;
    background: transparent;
    color: var(--system-message-text);
    box-shadow: none;
    text-align: center;
  }

  .message.system p {
    font-size: 0.86rem;
    font-weight: 500;
    text-transform: none;
  }

  .message p {
    margin: 0;
    line-height: 1.45;
    font-weight: 760;
    overflow-wrap: anywhere;
  }

  .file-message {
    display: inline-flex;
    align-items: center;
    gap: 10px;
    color: inherit;
    text-decoration: none;
  }

  .file-message > span:first-child {
    display: grid;
    width: 30px;
    height: 30px;
    place-items: center;
    border: 2px solid currentColor;
    font-weight: 950;
  }

  .file-message svg {
    width: 18px;
    height: 18px;
    fill: none;
    stroke: currentColor;
    stroke-linecap: square;
    stroke-linejoin: miter;
    stroke-width: 2.5;
  }

  .pending-file-icon svg {
    animation: file-spin 900ms linear infinite;
    transform-origin: center;
  }

  @keyframes file-spin {
    to {
      transform: rotate(360deg);
    }
  }

  .file-message strong,
  .file-message small {
    display: block;
  }

  .file-message strong {
    max-width: 28ch;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  .file-message small {
    margin-top: 2px;
    font-size: 0.76rem;
    font-weight: 750;
    opacity: 0.72;
  }

  .upload-progress {
    height: 8px;
    margin-top: 12px;
    border: 2px solid currentColor;
    background: color-mix(in oklch, currentColor 14%, transparent);
    overflow: hidden;
  }

  .upload-progress span {
    display: block;
    height: 100%;
    background: currentColor;
    transition: width 160ms cubic-bezier(0.16, 1, 0.3, 1);
  }

  .message-hint {
    position: absolute;
    right: -3px;
    top: 0;
    display: inline-flex;
    align-items: center;
    min-height: 28px;
    padding: 0 9px;
    border: 2px solid var(--copy-popover-border);
    border-radius: 0;
    background: var(--copy-popover-bg);
    color: var(--copy-popover-text);
    box-shadow: 3px 3px 0 var(--copy-popover-border);
    font-size: 0.72rem;
    font-weight: 950;
    pointer-events: none;
    transform: translateY(calc(-100% - 8px));
    text-transform: uppercase;
  }

  .message-form {
    padding: 18px;
    border-top: 4px solid var(--chat-border);
    background: var(--copy-note-bg);
  }

  .message-form input {
    min-height: 48px;
  }

  .file-input {
    position: absolute;
    width: 1px;
    height: 1px;
    margin: -1px;
    overflow: hidden;
    clip: rect(0, 0, 0, 0);
    clip-path: inset(50%);
    opacity: 0;
    pointer-events: none;
  }

  .file-input:disabled {
    opacity: 0;
  }

  .message-action {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    width: 48px;
    min-width: 48px;
    padding-inline: 0;
  }

  .message-action span {
    font-size: 1.18rem;
    line-height: 1;
  }

  .sr-only {
    position: absolute;
    width: 1px;
    height: 1px;
    padding: 0;
    margin: -1px;
    overflow: hidden;
    clip: rect(0, 0, 0, 0);
    white-space: nowrap;
    border: 0;
  }

  @media (max-width: 820px) {
    .app-shell {
      padding: 16px;
    }

    .topbar {
      align-items: flex-start;
    }

    .workspace {
      grid-template-columns: 1fr;
    }

    .chat-panel {
      grid-template-rows: auto minmax(360px, 58svh) auto;
    }

    .chat-window {
      padding: 18px;
    }

    .message {
      max-width: 86%;
    }
  }

  @media (max-width: 560px) {
    .app-shell {
      padding: 10px;
    }

    .brand small,
    .status-pill {
      display: none;
    }

    .peer-id-row,
    .input-action,
    .message-form {
      flex-direction: column;
    }

    .primary-button,
    .ghost-button {
      width: 100%;
    }
  }
</style>
