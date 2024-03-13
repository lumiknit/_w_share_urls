<script lang="ts">
  import "@picocss/pico";
  import pako from "pako";
  import toast, { Toaster } from "svelte-french-toast";

  import BsTrash from "svelte-icons-pack/bs/BsTrash";
  import BsPlus from "svelte-icons-pack/bs/BsPlus";

  import { onMount } from "svelte";
  import Icon from "svelte-icons-pack";

  let compressed: string = "";
  let urlList: string[] = [];
  let editMode: boolean = false;

  const addItem = () => {
    urlList = [...urlList, ""];
    toast.success("Added", {
      duration: 500,
    });
  };

  const removeItemAt = (i: number) => {
    urlList = urlList.filter((_, j) => i !== j);
    toast.success("Removed", {
      duration: 500,
    });
  };

  $: if (!editMode) {
    // Filter urlList to remove empty strings
    urlList = urlList.map((url) => url.trim()).filter((url) => url !== "");
    // Compress into base64
    compressed = compressList();
  }

  $: thisURL = (() => {
    // Get current protocol + host + path
    const url =
      window.location.protocol +
      "//" +
      window.location.host +
      window.location.pathname;
    // Add query param 'd'
    if (compressed !== "") {
      return url + "?d=" + compressed;
    }
    return url;
  })();

  // URL Modifier
  const splitProtocolAndURL = (url: string) => {
    // Split by protocol
    const protocolParts = url.split("://", 2);
    const protocol = protocolParts.length === 2 ? protocolParts[0] : "";
    url = protocolParts[protocolParts.length - 1];
    return [protocol, url];
  };

  const reduceURL = (url: string) => {
    // If the protocol is https, omit the protocol
    url = url.trim().replace(/^https?:\/\//, "");
    // Split by protocol
    const [protocol, url2] = splitProtocolAndURL(url);
    // Split by slash
    const parts = url2.split("/");
    // For the first part, if starts with www, omit it
    if (parts[0].startsWith("www.")) {
      parts[0] = parts[0].slice(3);
    }
    // If the first part ends with .com, omit it
    if (parts[0].endsWith(".com")) {
      parts[0] = parts[0].slice(0, -3);
    }
    // Join with slash
    url = parts.join("/");
    // Join with protocol
    if (protocol !== "") {
      url = protocol + "://" + url;
    }
    return url;
  };

  const expandURL = (url: string) => {
    // Split by protocol
    const [protocol, url2] = splitProtocolAndURL(url.trim());
    // Split by slash
    const parts = url2.split("/");
    // For the first part, if it's not a domain, add www
    if (parts[0].startsWith(".")) {
      parts[0] = "www" + parts[0];
    }
    if (parts[0].endsWith(".")) {
      parts[0] = parts[0] + "com";
    }
    // Join with slash
    url = parts.join("/");
    // Join with protocol
    if (protocol !== "") {
      url = protocol + "://" + url;
    } else {
      url = "https://" + url;
    }
    return url;
  };

  const compressList = () => {
    // If the protocol is https, omit the protocol
    const urls = urlList.map(reduceURL);

    // Join with newline
    const data = urls.join("\n");

    // Compress with gzip
    const deflated = pako.deflateRaw(data);

    // Convert to base64
    let base64 = btoa(String.fromCharCode(...new Uint8Array(deflated)));

    // Replace some characters to make it URL safe
    base64 = base64.replace(/\+/g, "-").replace(/\//g, "_").replace(/=/g, "");

    return base64;
  };

  const parseData = (d: string) => {
    // Convert given d from base64 to Uint8Array
    const bytes = atob(d);
    const uint8Array = new Uint8Array(bytes.length);
    for (let i = 0; i < bytes.length; i++) {
      uint8Array[i] = bytes.charCodeAt(i);
    }

    console.log(uint8Array);

    // Decompress with gzip
    const inflated = pako.inflateRaw(uint8Array, { to: "string" });
    urlList = inflated.split("\n").map(expandURL);
    console.log(urlList);
  };

  onMount(() => {
    // Get 'd' query param
    const urlParams = new URLSearchParams(window.location.search);
    const d = urlParams.get("d");
    if (d !== null) {
      compressed = d;
      parseData(compressed);
    }
  });
</script>

<Toaster position="bottom-center" />

<header class="container">
  <hgroup>
    <h1>Share URLs</h1>
  </hgroup>

  <fieldset>
    <label>
      <input
        name="terms"
        type="checkbox"
        role="switch"
        bind:checked={editMode}
      />
      Edit Mode
    </label>
  </fieldset>
</header>

<main class="container">
  {#if editMode}
    {#each urlList as url, i}
      <fieldset role="group">
        <input type="text" bind:value={url} />
        <button on:click={() => removeItemAt(i)}>
          <Icon src={BsTrash} />
        </button>
      </fieldset>
    {/each}
    <button class="w-100" on:click={addItem}>
      <Icon src={BsPlus} />
      Add
    </button>
  {:else}
    {#each urlList as url, i}
      <div class="linked-box-wrap">
        <a class="linked-box" href={url}>
          <img
            alt={"" + i}
            src={"https://www.google.com/s2/favicons?sz=256&sz=128&sz=64&sz=32&domain=" +
              url}
          />
          {url}
        </a>
      </div>
    {/each}

    <hr />

    To share, copy the below URL:<br />
    <a href={thisURL} target="_blank" rel="noopener">
      {thisURL}
    </a>
  {/if}
</main>

<style>
  .w-100 {
    width: 100%;
  }

  div.linked-box-wrap {
    display: inline-block;
    width: 25%;
    padding: 0.5rem;
  }

  a.linked-box {
    display: flex;
    position: relative;
    flex-direction: column;
    align-items: center;
    justify-content: start;
    word-wrap: break-word;
    word-break: break-all;
    font-size: 0.75rem;

    background-color: #8881;
    padding: 0.5rem;
    border-radius: 0.5rem;
    box-shadow: 0 0.25rem 0.5rem #0002;

    &:hover {
      background-color: #8884;
      box-shadow: 0 0.3rem 0.6rem #0002;
    }

    &:active {
      background-color: #8880;
      box-shadow: 0 0 0 #0002;
    }

    & > img {
      width: 75%;
    }
  }
</style>