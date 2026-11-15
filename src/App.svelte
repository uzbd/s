<script lang="ts">
  type Mode = "decode" | "encode";
  let mode = $state<Mode>("decode");
  let style = $state("mixed");
  let inputs = $state({ decode: "", encode: "" });
  let outputs = $state({ decode: "", encode: "" });
  let busy = $state({ decode: false, encode: false });
  let dict = $state("Loading dictionary…");
  let note = $state("");
  const ids = { decode: 0, encode: 0 };
  let seq = 0;

  const worker = new Worker(new URL("./worker.ts", import.meta.url), { type: "module" });
  worker.onmessage = (e: MessageEvent) => {
    const m = e.data;
    if (m.type === "ready") {
      dict = m.full ? "Full English dictionary loaded" : "Built-in dictionary only (full list unavailable)";
      run("decode");
      run("encode");
    } else if (m.type === "result" && m.id === ids[m.mode as Mode]) {
      outputs[m.mode as Mode] = m.text;
      busy[m.mode as Mode] = false;
    }
  };
  worker.postMessage({
    type: "init",
    words: new URL("words.txt", document.baseURI).href,
    common: new URL("common-words.txt", document.baseURI).href
  });

  function run(m: Mode) {
    const id = ++seq;
    ids[m] = id;
    busy[m] = true;
    worker.postMessage({ type: "run", id, mode: m, style: m === "encode" ? style : "", text: inputs[m] });
  }
  // Live: every change is sent to the worker; the newest request cancels older ones.
  $effect(() => run("decode"));
  $effect(() => run("encode"));

  function flash(t: string) { note = t; setTimeout(() => (note = ""), 2500); }
  async function copy() {
    if (!outputs[mode]) return flash("Nothing to copy yet.");
    try { await navigator.clipboard.writeText(outputs[mode]); flash("Copied to clipboard."); }
    catch { flash("Copy blocked by the browser. Select the text and copy it manually."); }
  }
  function download() {
    if (!outputs[mode]) return flash("Nothing to download yet.");
    const a = document.createElement("a");
    a.href = URL.createObjectURL(new Blob([outputs[mode]], { type: "text/plain;charset=utf-8" }));
    a.download = `${mode}d-text.txt`;
    a.click();
    setTimeout(() => URL.revokeObjectURL(a.href), 1000);
  }
  function sendOver() {
    const other: Mode = mode === "decode" ? "encode" : "decode";
    inputs[other] = outputs[mode];
    mode = other;
  }
  async function openFile(f?: File) { if (f) inputs[mode] = await f.text(); }
</script>

<main>
  <h1>Smart Text Decoder</h1>
  <p class="sub">Clean up dotted, spaced and symbol-scrambled text, or scramble your own. Live, private, no size limit.</p>
  <div class="tabs">
    <button class:on={mode === "decode"} onclick={() => (mode = "decode")}>Decode</button>
    <button class:on={mode === "encode"} onclick={() => (mode = "encode")}>Encode</button>
  </div>

  <div class="grid">
    <section class="card">
      <div class="head"><span>{mode === "decode" ? "Scrambled text" : "Normal text"}</span><small>{inputs[mode].length.toLocaleString()} chars</small></div>
      <textarea
        bind:value={inputs[mode]}
        spellcheck="false"
        placeholder={mode === "decode" ? "he. ll o.wo .rl d\nhe$llo.w$orl...d" : "Type or paste normal text…"}
        ondrop={(e) => { const f = e.dataTransfer?.files?.[0]; if (f) { e.preventDefault(); openFile(f); } }}
      ></textarea>
      <div class="bar">
        <label class="b">Open .txt<input type="file" accept=".txt,text/plain" onchange={(e) => openFile(e.currentTarget.files?.[0])} /></label>
        <button class="b" onclick={() => (inputs[mode] = "")}>Clear</button>
        {#if mode === "encode"}
          <select bind:value={style} aria-label="Scramble style">
            <option value="mixed">Mixed noise</option>
            <option value="dots">Dots</option>
            <option value="symbols">Symbols</option>
            <option value="spaced">Spaced letters</option>
          </select>
        {/if}
      </div>
    </section>

    <section class="card">
      <div class="head"><span>{mode === "decode" ? "Readable text" : "Scrambled text"}</span><small>{busy[mode] ? "working…" : `${outputs[mode].length.toLocaleString()} chars`}</small></div>
      <textarea readonly value={outputs[mode]}></textarea>
      <div class="bar">
        <button class="b" onclick={copy}>Copy</button>
        <button class="b" onclick={download}>Download .txt</button>
        <button class="b" onclick={sendOver}>Send to {mode === "decode" ? "Encode" : "Decode"}</button>
      </div>
    </section>
  </div>
  <p class="status" aria-live="polite"><b>{note}</b> {note ? "·" : ""} {dict}</p>
</main>
