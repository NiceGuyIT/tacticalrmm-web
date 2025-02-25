<template>
  <q-dialog
    ref="dialogRef"
    maximized
    @hide="onDialogHide"
    @show="loadEditor"
    @before-hide="unloadEditor"
  >
    <q-card class="q-dialog-plugin">
      <q-bar>
        <span class="q-pr-sm">{{ title }}</span>
        <q-btn
          v-if="!snippet && openAIEnabled"
          :disable="loading"
          dense
          size="xs"
          label="Generate Script"
          color="primary"
          no-caps
          @click="generateScriptOpenAI"
        />
        <q-space />
        <q-btn dense flat icon="close" v-close-popup>
          <q-tooltip class="bg-white text-primary">Close</q-tooltip>
        </q-btn>
      </q-bar>
      <div class="row">
        <q-input
          :rules="[(val: string) => !!val || '*Required']"
          class="q-pa-sm col-4"
          v-model="snippet.name"
          label="Name"
          filled
          dense
        />
        <q-select
          v-model="snippet.shell"
          :options="shellOptions"
          class="q-pa-sm col-2"
          label="Shell Type"
          options-dense
          filled
          dense
          emit-value
          map-options
        />
        <q-input
          class="q-pa-sm col-6"
          filled
          dense
          v-model="snippet.desc"
          label="Description"
        />
      </div>

      <div
        ref="snippetEditor"
        :style="{ height: `${$q.screen.height - 132}px` }"
      ></div>

      <q-card-actions align="right">
        <q-btn dense flat label="Cancel" v-close-popup />
        <q-btn
          :loading="loading"
          dense
          flat
          label="Save"
          color="primary"
          @click="submit"
        />
      </q-card-actions>
    </q-card>
  </q-dialog>
</template>

<script setup lang="ts">
// composable imports
import { ref, watch, reactive, computed } from "vue";
import { useStore } from "vuex";
import { useQuasar } from "quasar";
import { generateScript } from "@/api/core";
import { useDialogPluginComponent } from "quasar";
import { saveScriptSnippet, editScriptSnippet } from "@/api/scripts";
import { notifySuccess } from "@/utils/notify";

// ui imports
import * as monaco from "monaco-editor";

// types
import type { ScriptSnippet } from "@/types/scripts";

// static data
import { shellOptions } from "@/composables/scripts";

// props
const props = defineProps<{ snippet?: ScriptSnippet }>();

// emits
defineEmits([...useDialogPluginComponent.emits]);

// quasar dialog setup
const { dialogRef, onDialogHide, onDialogOK } = useDialogPluginComponent();

// setup quasar
const $q = useQuasar();

// setup store
const store = useStore();
const openAIEnabled = computed(() => store.state.openAIIntegrationEnabled);

// snippet form logic
const snippet: ScriptSnippet = props.snippet
  ? reactive(Object.assign({}, props.snippet))
  : reactive({ name: "", code: "", shell: "powershell" });
const loading = ref(false);

const title = computed(() => {
  if (props.snippet) {
    return `Editing ${snippet.name}`;
  } else {
    return "Adding New Script Snippet";
  }
});

// convert highlighter language to match what ace expects
const lang = computed(() => {
  if (snippet.shell === "cmd") return "bat";
  else if (snippet.shell === "powershell") return "powershell";
  else if (snippet.shell === "python") return "python";
  else if (snippet.shell === "shell") return "shell";
  else return "";
});

async function submit() {
  loading.value = true;
  try {
    const result = props.snippet
      ? await editScriptSnippet(snippet)
      : await saveScriptSnippet(snippet);
    onDialogOK();
    notifySuccess(result);
  } catch (e) {
    console.error(e);
  }

  loading.value = false;
}

const snippetEditor = ref<HTMLElement | null>(null);
let editor: monaco.editor.IStandaloneCodeEditor;

function loadEditor() {
  var modelUri = monaco.Uri.parse("model://snippet"); // a made up unique URI for our model
  var model = monaco.editor.createModel(snippet.code, lang.value, modelUri);

  const theme = $q.dark.isActive ? "vs-dark" : "vs-light";

  // eslint-disable-next-line @typescript-eslint/no-non-null-assertion
  editor = monaco.editor.create(snippetEditor.value!, {
    automaticLayout: true,
    model: model,
    theme: theme,
  });

  editor.onDidChangeModelContent(() => {
    snippet.code = editor.getValue();
  });

  // watch for changes in language
  watch(lang, () => {
    monaco.editor.setModelLanguage(model, lang.value);
  });
}

function unloadEditor() {
  editor.getModel()?.dispose();
  editor.dispose();
  onDialogHide();
}

function generateScriptOpenAI() {
  $q.dialog({
    title: "Ask ChatGPT what you need!",
    prompt: {
      model: `${lang.value} code that `,
      type: "text",
    },
    cancel: true,
    persistent: true,
  }).onOk(async (data) => {
    const completion = await generateScript({
      prompt: data,
    });
    snippet.code = completion;
  });
}
</script>
