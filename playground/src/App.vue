<template>
<div id="root">
	<Settings v-if="showSettings" @exit="showSettings = false" />
	<h1>AiScript (v{{ AISCRIPT_VERSION }}) Playground<button id="show-settings-button" @click="showSettings = true">Settings</button></h1>
	<div id="grid1">
		<div id="editor" class="container">
			<header>Input<div class="actions"><button @click="setCode">FizzBuzz</button></div></header>
			<div>
				<PrismEditor class="code" v-model="script" :highlight="highlighter" :line-numbers="false"/>
			</div>
			<footer>
				<span v-if="syntaxErrorMessage" class="syntaxError">{{ syntaxErrorMessage }}</span>
				<div class="actions"><button @click="run">RUN</button></div>
			</footer>
		</div>
		<div id="logs" class="container">
			<header>
				Output
				<div v-if="paused" class="actions"><button @click="interpreter.unpause(), paused = false">Unpause</button></div>
				<div v-else class="actions"><button @click="interpreter.pause(), paused = true">Pause</button></div>
			</header>
			<div>
				<div v-for="log in logs" class="log" :key="log.id" :class="[{ print: log.print }, log.type]"><span class="type">{{ log.type }}</span> {{ log.text }}</div>
			</div>
		</div>
	</div>
	<div id="grid2">
		<div id="ast" class="container">
			<header>AST</header>
			<div>
				<pre>{{ JSON.stringify(ast, (_key, value) => {
					if (value instanceof Map) {
						return Object.fromEntries(value);
					}
					return value;
				}, '\t') }}</pre>
			</div>
		</div>
		<div id="bin" class="container">
			<header>Bytecode</header>
			<div>
			</div>
		</div>
		<div id="debugger" class="container">
			<header>Debugger</header>
			<div>
			</div>
		</div>
	</div>
</div>
</template>

<script setup>
import { ref, watch } from 'vue';
import { AISCRIPT_VERSION, Interpreter, Parser, utils } from '@syuilo/aiscript';
import { std } from '@syuilo/aiscript/interpreter/lib/std';

import { PrismEditor } from 'vue-prism-editor';
import 'vue-prism-editor/dist/prismeditor.min.css';
import 'prismjs';
import { highlight, languages } from 'prismjs/components/prism-core';
import 'prismjs/components/prism-clike';
import 'prismjs/components/prism-javascript';
import 'prismjs/themes/prism-okaidia.css';
import Settings, { settings } from './Settings.vue';

const script = ref(window.localStorage.getItem('script') || '<: "Hello, AiScript!"');

const ast = ref(null);
const logs = ref([]);
const syntaxErrorMessage = ref(null);
const showSettings = ref(false);
const paused = ref(false);

watch(script, () => {
	window.localStorage.setItem('script', script.value);
	try {
		ast.value = Parser.parse(script.value);
		syntaxErrorMessage.value = null;
	} catch (e) {
		syntaxErrorMessage.value = e.message;
		console.error(e.info);
		return;
	}
}, {
	immediate: true
});

const setCode = () => {
	script.value = `for (let i, 100) {
  <: if (i % 15 == 0) "FizzBuzz"
    elif (i % 3 == 0) "Fizz"
    elif (i % 5 == 0) "Buzz"
    else i
}`;
};

let interpreter = null;
const run = async () => {
	logs.value = [];

	interpreter?.abort();
	paused.value = false;
	interpreter = new Interpreter({}, {
		in: (q) => {
			return new Promise(ok => {
				const res = window.prompt(q);
				ok(res);
			});
		},
		out: (value) => {
			logs.value.push({
				id: Math.random(),
				type: value.type,
				text: value.type === 'str' || value.type === 'num' ? value.value : utils.valToString(value),
				print: true
			});
		},
		err: (e) => {
			window.alert(e.toString());
		},
		log: (type, params) => {
			switch (type) {
				case 'end': logs.value.push({
					id: Math.random(),
					text: utils.valToString(params.val, true),
					print: false
				}); break;
				default: break;
			}
		},
		irqRate: settings.value.irqRate,
		irqSleep: settings.value.irqSleep,
	});

	try {
		await interpreter.exec(ast.value);
	} catch (e) {
		console.error(e);
		window.alert('Internal Error: ' + e);
	}
}

const highlighter = code => {
	return highlight(code, languages.js, 'javascript');
};
</script>

<style>
:root {
	--borderThickness: 1px;
}

* {
	font-family: Fira code, Fira Mono, Consolas, Menlo, Courier, monospace;
}

html {
	background: #171717;
	color: #fff;
	tab-size: 2;
}

body {
	margin: 0;
	padding: 0;
}

pre {
	margin: 0;
}

#root {
	display: flex;
	flex-direction: column;
	height: 100vh;
}
#root > h1 {
	font-size: 1.5em;
	margin: 16px 16px 0 16px;
	display: flex;
}

#show-settings-button {
	margin-left: auto;
}

#grid1 {
	box-sizing: border-box;
	flex: 1;
	display: grid;
	grid-template-columns: 1fr 1fr;
	grid-template-rows: 1fr;
	grid-gap: 16px;
	padding: 16px 16px 16px 16px;
	min-height: 0;
}
#grid1 > * {
	min-height: 0;
}
#grid2 {
	box-sizing: border-box;
	flex: 1;
	display: grid;
	grid-template-columns: 1fr 1fr 1fr;
	grid-template-rows: 1fr;
	grid-gap: 16px;
	padding: 0 16px 16px 16px;
	min-height: 0;
}
#grid2 > * {
	min-height: 0;
}

#editor {
}
#editor > .code {
	box-sizing: border-box;
	padding: 16px;
}
#editor .syntaxError {
	color: #f00;
}

#logs .log .type {
	opacity: 0.5;
	color: #fff;
}
#logs .log:not(.print) {
	opacity: 0.7;
}
#logs .log.num {
	color: #0ff;
}
#logs .log.str {
	color: #ff0;
}

#ast {

}

.container {
	display: flex;
	flex-direction: column;
	border: solid var(--borderThickness) #555;
	border-radius: 8px;
	background: #202020;
}
.container > header {
	display: flex;
	padding: 8px 16px;
	border-bottom: dashed var(--borderThickness) #555;
	font-weight: bold;
}
.container > header > .actions {
	margin-left: auto;
}
.container > div {
	flex: 1;
	overflow: auto;
	padding: 16px;
}
.container > footer {
	display: flex;
	padding: 8px 16px;
	border-top: dashed var(--borderThickness) #555;
}
.container > footer > .actions {
	margin-left: auto;
}
</style>
