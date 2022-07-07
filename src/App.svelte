<script lang="ts">
  import { readDir, renameFile, BaseDirectory } from "@tauri-apps/api/fs";
  import { open } from "@tauri-apps/api/dialog";
  import { confirm } from "@tauri-apps/api/dialog";
  import { join } from "@tauri-apps/api/path";
  import Help from "./lib/Help.svelte";

  // 1 - old fileName, 2 - new one, 3 - file extension
  type FileDescriptor = [string, string?, string?];
  const COUNTER_REGEX = "{c}";
  const ORIGNAME_REGEX = "{o}";
  let readyToRename: boolean = false;

  let files: FileDescriptor[] = [];
  let fileRegex: string = "";
  let selectedDir: string = "";

  async function readFilesInDirectory(folderPath: string): Promise<void> {
    const dirEntries = await readDir(folderPath, { recursive: false });
    files = dirEntries
      .filter((entry) => !entry.children)
      .map((entry) => {
        const result = entry.name.match(/(^.+)(\.\w+$)/i);
        if (result === null) {
          return [entry.name, null, null];
        }
        return [result[1], null, result[2]];
      });
  }

  async function openFolder(): Promise<string | null> {
    const selected = await open({
      multiple: false,
      directory: true,
    });
    if (typeof selected === "string") {
      return selected;
    } else {
      return null;
    }
  }

  function prepareRenamingFiles() {
    files.forEach((descriptor) => (descriptor[1] = null));
    let newFileNameRegEx = fileRegex;
    if (
      !newFileNameRegEx.includes(COUNTER_REGEX) &&
      !newFileNameRegEx.includes(ORIGNAME_REGEX)
    ) {
      newFileNameRegEx = newFileNameRegEx.concat(`_${COUNTER_REGEX}`);
    }

    files.forEach((descriptor, index) => {
      let fileNewName = newFileNameRegEx
        .replaceAll("{o}", descriptor[0])
        .replaceAll("{c}", (index + 1).toString());

      if (fileNewName.length > 200) {
        fileNewName = fileNewName.substring(0, 200);
      }

      descriptor[1] = fileNewName;
    });
    files = files;
    readyToRename = true;
  }

  async function renameFiles() {
    const confirmed = await confirm(
      "Это действие нельзя отменить. Продолжить?",
      { title: "Renamer", type: "warning" }
    );
    if (confirmed) {
      await Promise.all(
        files.map(async (descriptor) => {
          const oldFilePath = await join(
            selectedDir,
            `${descriptor[0]}${descriptor[2]}`
          );
          const newFilePath = await join(
            selectedDir,
            `${descriptor[1]}${descriptor[2]}`
          );
          return renameFile(oldFilePath, newFilePath);
        })
      );
      await readFilesInDirectory(selectedDir);
      fileRegex = "";
      readyToRename = false;
    }
  }
</script>

<input type="checkbox" id="my-modal" class="modal-toggle" />
<div class="modal">
  <div class="modal-box">
    <h3 class="font-bold text-lg">Справка:</h3>
    <p class="py-4">&#123; c &#125; - подставляет счетчик</p>
    <p class="py-4">
      &#123; o &#125; - подставляет оригинальное название файла
    </p>
    <div class="modal-action">
      <label for="my-modal" class="btn">ОК</label>
    </div>
  </div>
</div>

<main class="relative">
  <div
    class="mb-4 sticky top-0 z-40 navbar bg-base-100 flex flex-wrap justify-center space-x-4 bg-blue-100"
  >
    <input
      bind:value={fileRegex}
      on:input={() => {
        readyToRename = false;
      }}
      type="text"
      placeholder="Новое имя файлов"
      class="input input-bordered w-full max-w-xs input-md"
    />
    <button
      on:click={async () => {
        const folderPath = await openFolder();
        if (folderPath !== null) {
          selectedDir = folderPath;
          await readFilesInDirectory(selectedDir);
        }
      }}
      class="btn btn-primary">Открыть папку</button
    >
    {#if !readyToRename}
      <button
        on:click={prepareRenamingFiles}
        disabled={!fileRegex}
        class="btn btn-warning">Переименовать</button
      >
    {:else}
      <button on:click={renameFiles} class="btn btn-success">Применить</button>
    {/if}
    <label
      for="my-modal"
      class="btn btn-circle btn-xs btn-outline btn-primary justify-self-end"
      >?</label
    >
  </div>

  <div class="overflow-y-auto overflow-x-hidden mt-4 mb-12">
    <table class="table table-compact w-full table-fixed">
      <thead>
        <tr class="bg-indigo-100">
          <th class="w-10" />
          <th>Имя файла</th>
          <th>Новое имя</th>
        </tr>
      </thead>
      <tbody>
        {#each files as descriptor, i}
          <tr>
            <td class="overflow-hidden border border-slate-300 font-bold"
              >{i + 1}</td
            >
            <td
              class="overflow-hidden text-ellipsis border border-slate-300"
              data-bs-toggle="tooltip"
              data-bs-placement="right"
              title={descriptor[0]}
            >
              {descriptor[2] === null
                ? descriptor[0]
                : descriptor[0] + descriptor[2]}
            </td>
            {#if descriptor[1] != null}
              <td class="overflow-hidden text-ellipsis border border-slate-300">
                {descriptor[2] === null
                  ? descriptor[1]
                  : descriptor[1] + descriptor[2]}
              </td>
            {/if}
          </tr>
        {/each}
      </tbody>
    </table>
  </div>

  <footer
    class="footer footer-center fixed bottom-0 p-4 bg-blue-400 text-base-content"
  >
    <div>
      <p class="text-white">
        Copyright © {new Date().getFullYear()} - Все права принадлежат Apheliont
      </p>
    </div>
  </footer>
</main>
