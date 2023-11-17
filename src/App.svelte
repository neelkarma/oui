<script>
  import { micromark } from "micromark";
  import LoadingDots from "./lib/LoadingDots.svelte";

  let hostname = "http://localhost:11434";
  let selectedModel;
  let prompt = "";
  let thinking = false;
  let modelContext;
  let aborter;

  $: generateEndpoint = `${hostname}/api/generate`;
  $: modelsEndpoint = `${hostname}/api/tags`;

  let models = [];

  let chat = [];

  $: {
    fetch(modelsEndpoint)
      .then((response) => response.json())
      .then((data) => {
        models = data.models.map((model) => model.name);
        if (models) selectedModel = models[0];
      })
      .catch((error) => {
        console.error(error);
      });
  }

  const handleSend = async () => {
    if (thinking) {
      aborter.abort();
      thinking = false;
      return;
    }

    if (!prompt) return;

    thinking = true;
    chat.push({
      author: "user",
      text: prompt,
    });
    chat.push({
      author: "model",
      text: "",
    });
    chat = chat;
    const p = prompt;
    prompt = "";

    aborter = new AbortController();
    const res = await fetch(generateEndpoint, {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
      },
      signal: aborter.signal,
      body: JSON.stringify({
        model: selectedModel,
        prompt: p,
        context: modelContext,
      }),
    });

    const reader = res.body
      .pipeThrough(new TextDecoderStream())
      .pipeThrough(
        new TransformStream({
          start() {},
          async transform(chunk, controller) {
            controller.enqueue(JSON.parse(chunk));
          },
          flush() {},
        })
      )
      .getReader();

    while (true) {
      const { done: fetchDone, value } = await reader.read();
      chat[chat.length - 1].text += value.response;
      console.log(value);
      if (fetchDone || value.done) {
        modelContext = value.context;
        break;
      }
    }
    thinking = false;
  };
</script>

<div class="top-bit">
  {#each chat as message}
    <div>
      <p class="author">&gt; {message.author}</p>
      {#if message.author === "model" && !message.text && thinking}
        <LoadingDots />
      {:else}
        {@html micromark(message.text)}
      {/if}
    </div>
  {/each}

  {#if !chat.length}
    <div class="config-panel">
      <label>
        Hostname
        <input type="text" bind:value={hostname} />
      </label>

      <label>
        Model Name
        <select bind:value={selectedModel}>
          {#each models as model}
            <option value={model}>{model}</option>
          {/each}
        </select>
      </label>
    </div>
  {/if}
</div>

<form class="input-bar" on:submit|preventDefault={handleSend}>
  <input
    type="text"
    placeholder="Type a message..."
    disabled={thinking}
    bind:value={prompt}
  />
  <button disabled={!thinking && !prompt}>
    {#if thinking}
      <span>Stop</span>
    {:else}
      <span>Send</span>
    {/if}
  </button>
  {#if chat.length}
    <button
      type="button"
      on:click={() => {
        chat = [];
        modelContext = undefined;
      }}>Reset</button
    >
  {/if}
</form>

<style>
  input,
  select,
  button {
    padding: 8px;
    font-size: 0.95rem;
    border: 1px solid #555;
    background-color: #222;
    border-radius: 8px;
  }

  label {
    display: flex;
    align-items: center;
    gap: 8px;
  }

  label > input,
  label > select {
    flex: 1;
  }
  .config-panel {
    display: flex;
    flex-direction: column;
    gap: 5px;
    padding: 8px;
    border: 1px solid #555;
    border-radius: 8px;
    background-color: #333;
  }

  .input-bar {
    display: flex;
    position: relative;
    gap: 5px;
  }

  .input-bar > input {
    padding: 12px;
    flex: 1;
  }

  .top-bit {
    flex: 1;
  }

  .author {
    color: #888;
  }
</style>
