<template>
<div ref="term"></div>
</template>

<script>
    import { Terminal } from 'xterm';
    import LocalEchoController from 'local-echo';
    import { FitAddon } from 'xterm-addon-fit';
    export default {
        name: "Terminal",
	    mounted() {
            const term = new Terminal();
            term.open(this.$refs.term);
            const localEcho = new LocalEchoController();
            const fitAddon = new FitAddon();
            term.loadAddon(localEcho);
            term.loadAddon(fitAddon);
            fitAddon.fit();
            //term.write('Hello from \x1B[1;3;31mxterm.js\x1B[0m $ ');
            localEcho.read(">")
                .then(input => alert(`User entered: ${input}`))
                .catch(error => alert(`Error reading: ${error}`));
        }
    }
</script>

<style scoped>
	@import '~xterm/css/xterm.css';
</style>
