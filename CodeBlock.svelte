<script>
	import { onMount } from 'svelte';
	import hljs from 'highlight.js/lib/core';
	import javascript from 'highlight.js/lib/languages/javascript';
	import python from 'highlight.js/lib/languages/python';
	import java from 'highlight.js/lib/languages/java';
	import csharp from 'highlight.js/lib/languages/csharp';
	import cpp from 'highlight.js/lib/languages/cpp';
	import rust from 'highlight.js/lib/languages/rust';
	import 'highlight.js/styles/github-dark.css';

	let code = 'print("Hello World!")\n';
	let language = 'python';
	let highlighted = '';
	let codeblock;
	let codeblockContainer;
	let debounceTimer;
	let enterKeyPressed = false;
	let stdout = '';
	let pyodide;

	hljs.registerLanguage('javascript', javascript);
	hljs.registerLanguage('python', python);
	hljs.registerLanguage('cpp', cpp);
	hljs.registerLanguage('csharp', csharp);
	hljs.registerLanguage('java', java);
	hljs.registerLanguage('rust', rust);

	function saveCaretPosition(element) {
		const selection = window.getSelection();
		if (selection.rangeCount > 0) {
			const range = selection.getRangeAt(0);
			const preCaretRange = range.cloneRange();
			preCaretRange.selectNodeContents(element);
			preCaretRange.setEnd(range.endContainer, range.endOffset);
			return preCaretRange.toString().length + (enterKeyPressed ? 1 : 0);
		}
		return 0;
	}

	function restoreCaretPosition(element, position) {
		const selection = window.getSelection();
		const range = document.createRange();
		let charCount = 0;

		function traverseNodes(currentNode) {
			if (currentNode.nodeType === Node.TEXT_NODE) {
				const nextCharCount = charCount + currentNode.length;
				if (position >= charCount && position <= nextCharCount) {
					range.setStart(currentNode, position - charCount);
					range.setEnd(currentNode, position - charCount);
					// console.log(currentNode, position - charCount);
					return true;
				}
				charCount = nextCharCount;
			} else {
				for (let i = 0; i < currentNode.childNodes.length; i++) {
					if (traverseNodes(currentNode.childNodes[i])) {
						return true;
					}
				}
			}
			return false;
		}

		traverseNodes(element);
		selection.removeAllRanges();
		selection.addRange(range);
	}

	// useful utility func
	function getCurrentLine(element, cursorPosition) {
		const text = element.textContent;
		const lineStart = text.lastIndexOf('\n', cursorPosition - 1) + 1;
		const lineEnd = text.indexOf('\n', cursorPosition);
		return text.substring(lineStart, lineEnd !== -1 ? lineEnd : text.length);
	}

	function updateHighlight() {
		if (!codeblock) return;

		const caretPosition = saveCaretPosition(codeblock);

		if (language === 'auto') {
			const result = hljs.highlightAuto(code);
			highlighted = result.value;
		} else {
			highlighted = hljs.highlight(code, { language }).value;
		}

		codeblock.innerHTML = highlighted;

		restoreCaretPosition(codeblock, caretPosition);
	}

	function handleInput(event) {
		code = event.target.innerText;
		clearTimeout(debounceTimer);

		debounceTimer = setTimeout(() => {
			updateHighlight();
		}, 200);
	}

	function handleLanguageChange(event) {
		language = event.target.value;
		updateHighlight();
	}

	function handleKeyPress(event) {
		if (event.key === 'Enter') {
			event.preventDefault();
			const range = window.getSelection()?.getRangeAt(0);
			const br = document.createElement('br');
			range?.insertNode(br);
			range?.setStartAfter(br);
			enterKeyPressed = true;
			return;
		} else if (event.ctrlKey && event.key === 's') {
			event.preventDefault();
		}
		enterKeyPressed = false;
	}

	onMount(() => {
		updateHighlight();
	});

	async function runPythonCode(pyodide, code) {
		try {
			// capture stdout
			pyodide.runPython(`
				import io, sys
				sys.stdout = io.StringIO()
			`);

			// run code
			let result = await pyodide.runPythonAsync(code);

			// capture stdout
			let stdout = pyodide.runPython('sys.stdout.getvalue()');

			// reset stdout
			pyodide.runPython('sys.stdout = sys.__stdout__');

			return { stdout: stdout, result: result };
		} catch (error) {
			console.error('Python Error:', error);
			return { error: error.message };
		}
	}

	function runJavaScriptCode(jsCode) {
		const iframe = document.createElement('iframe');
		iframe.style.display = 'none';
		document.body.appendChild(iframe);

		const iframeWindow = iframe.contentWindow;
		const script = iframeWindow.document.createElement('script');
		script.type = 'text/javascript';
		script.innerHTML = `
            const capturedOutput = [];
            const originalLog = console.log;
            console.log = function (...args) {
                capturedOutput.push(args.join(' '));
                originalLog.apply(console, args);
            };

            try {
                parent.postMessage({ stdout: eval(${JSON.stringify(jsCode)}), logs: capturedOutput }, '*');
            } catch (e) {
                parent.postMessage({ error: e.message }, '*');
            } finally {
                console.log = originalLog;
            }
        `;

		window.addEventListener('message', function handleMessage(event) {
			if (event.source === iframeWindow) {
				if (event.data.stdout !== undefined) {
					stdout = String(event.data.stdout);
				} else if (event.data.error !== undefined) {
					stdout = String(event.data.error);
				} else if (event.data.logs !== undefined) {
					stdout = event.data.logs.join('\n');
				}
				window.removeEventListener('message', handleMessage);
				document.body.removeChild(iframe);
			}
		});

		iframeWindow.document.body.appendChild(script);
	}

	async function runCode() {
		stdout = '';
		if (language === 'python') {
			if (!pyodide) pyodide = await loadPyodide();
			let output = await runPythonCode(pyodide, code);
			if (output.stdout) {
				stdout = output.stdout;
			} else if (output.error) {
				stdout = output.error;
			}

			console.log('Python stdout:', output.stdout);
			console.log('Python result:', output.result);
		} else if (language === 'javascript') {
			runJavaScriptCode(code);
		}
	}
</script>

<div class="code-block-container" bind:this={codeblockContainer}>
	<select class="code-block-select" bind:value={language} on:change={handleLanguageChange}>
		<option value="auto" class="grayed-out-entries">Auto</option>
		<option value="javascript">JavaScript</option>
		<option value="python">Python</option>
		<option value="java" class="grayed-out-entries">Java</option>
		<option value="cpp" class="grayed-out-entries">C++</option>
		<option value="csharp" class="grayed-out-entries">C#</option>
		<option value="rust" class="grayed-out-entries">Rust</option>
	</select>
	<pre
		class="code-block"
		spellcheck="false"
		contenteditable="true"
		on:input|stopPropagation={handleInput}
		on:keydown={handleKeyPress}
		on:keypress|stopPropagation
		on:keyup|stopPropagation
		bind:this={codeblock}><code class={language}></code></pre>
	{#if language === 'python' || language === 'javascript'}
		<button class="run-button" on:click={runCode}>
			<svg
				xmlns="http://www.w3.org/2000/svg"
				fill="var(--voithos-silver)"
				viewBox="0 0 24 24"
				width="14"
				height="14">
				<path d="M19.749,9.464,5,.048V23.989L19.743,14.54a3,3,0,0,0,.006-5.076Z" /></svg
			>
		</button>
	{/if}
</div>
{#if stdout.length > 0}
	<div class="stdout-wrapper">
		<pre class="standard-output">{stdout}</pre>
	</div>
{/if}

<style>
  .code-block-container {
  	position: relative;
  	padding-top: 20px;
  	padding-bottom: 20px;
  }
  .code-block {
  	background-color: #2e2e2e;
  	padding: 1em;
  	border-radius: 4px;
  	overflow-x: auto;
  	outline: none;
  	min-height: 50px;
  	margin: 0;
  	font-family: 'Courier New', Courier, monospace;
  }
  code {
  	outline: none;
  }
  .code-block-select {
  	position: absolute;
  	top: 10px;
  	right: 15px;
  	color: #ffffff;
  	border-radius: 3px;
  }
  
  .code-block-select option {
  	background-color: #4b4b4b;
  	color: #ffffff;
  }
  
  .code-block-select .grayed-out-entries {
  	background-color: #000000;
  	color: #a0a0a0;
  }
  
  .run-button {
  	position: absolute;
  	bottom: 10px;
  	right: 17px;
  	opacity: 0.5;
  }
  
  .stdout-wrapper {
  	margin-top: 20px;
  	padding-top: 20px;
  	padding-bottom: 10px;
  }
  
  .standard-output {
  	font-family: 'Courier New', Courier, monospace;
  	background-color: #2e2e2e;
  	padding: 1em;
  	border-radius: 4px;
  	overflow-x: auto;
  }
</style>
