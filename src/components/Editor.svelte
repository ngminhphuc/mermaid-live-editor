<script>
import { codeStore, updateCodeStore } from '../code-store.js';
import { codeErrorStore } from '../code-error-store.js';
import { onMount } from 'svelte';
import {push, pop, replace} from 'svelte-spa-router'
import { Base64 } from 'js-base64'
// import mermaid from '@mermaid-js/mermaid';
import mermaid from '@mermaid';
import Error from './Error.svelte';
import { initEditor } from './editor-utils';
import 'monaco-editor/esm/vs/editor/browser/controller/coreCommands.js';
import 'monaco-editor/esm/vs/editor/contrib/find/findController.js';
import * as monaco from 'monaco-editor/esm/vs/editor/editor.api.js';

export let code = '';
export let conf = { theme: 'default' };
let edit;
export let error = false;

let decorations = [];
const decArr = [];
let editorElem = null;
const handleCodeUpdate = code => {
	try {
		mermaid.parse(code);
		let newState = { code, mermaid: conf };
		updateCodeStore(newState);
		decArr.forEach(decor => {
			edit.deltaDecorations(decor, []);
		});

		codeErrorStore.set(undefined);
		const model = edit.getModel();
	} catch(e) {
		if(e) {
			codeErrorStore.set(e);
			console.log('Error in parsed', e.hash);
			const str = JSON.stringify({ code: code, mermaid: conf });
			push('/edit/' + Base64.encode(str))
			const l = e.hash.line;
			decArr.push(edit.deltaDecorations([], [
				{ range: new monaco.Range(e.hash.loc.first_line,e.hash.loc.last_line,e.hash.loc.first_column,e.hash.loc.last_column), options: { inlineClassName: 'myInlineDecoration' }}]
			));
		}
	}
};

const unsubscribe = codeStore.subscribe( state => {
	if(editorElem === null) {
		console.log('Starting stuff', document.getElementById('editor'));
		editorElem = document.getElementById('editor');
	}
	if(!code && state) {
		code = state.code;
	}
	if(state) {
		conf = state.mermaid;
	}
	if(!edit && code && (editorElem !== null)) {
		console.log('creatinf editor');
		edit = monaco.editor.create(editorElem, {
			value: [
				code,
			].join('\n'),
			theme: 'myCoolTheme',
			language: 'mermaid'
		});

		let decorations = [];
		edit.onDidChangeModelContent(function (e) {
			const code = edit.getValue();
			handleCodeUpdate(code);
		});
		handleCodeUpdate(code);
	}
});

const unsubscribeError = codeErrorStore.subscribe( _error => {
	if(_error) {
		error = true;
	} else {
		error = false;
	}
});

initEditor(monaco);

onMount(async () => {
	console.log('Mounting editor');
	// editorElem = document.querySelector('#editor')
	self.MonacoEnvironment = {
		getWorkerUrl: function (moduleId, label) {
			return './editor.worker.bundle.js';
		}
	}
});

	// export let name;
	// export let params = {};
</script>

<style>
	#editor-container {}
	#editor {
		width: 100%;
		height: 400px;
		max-height: 300px;
		flex: 1;
	}
</style>

<div id="editor-container">
	<div id="editor"> </div>
	{#if error}
	<Error errorText="Syntax Error"/>
	{/if}
</div>
