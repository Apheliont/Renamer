<script lang="ts">
  import { WebviewWindow } from "@tauri-apps/api/window";

  function openHelpWindow() {
    const webview = new WebviewWindow("Help", {
      url: "/windows/help.html",
      title: "Справка",
      height: 400,
      width: 400,
    });
  }
</script>

<button
  on:click={openHelpWindow}
  class="btn btn-circle btn-xs btn-outline btn-primary justify-self-end"
>
  ?
</button>
