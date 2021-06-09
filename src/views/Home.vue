<template>
  <v-app  class="home">
    <v-system-bar  color="primary"  class="titlebar text-center" dark>
      <v-spacer></v-spacer>Scheme It - {{$store.state.file.name||""}}
      <v-spacer></v-spacer></v-system-bar>
    <v-system-bar  class="grey darken-4" window dark><v-breadcrumbs style="padding-top: 0;padding-bottom: 0"
                              dark
                              :items="pathBreakdown">
      <template v-slot:divider>
        <v-icon>mdi-chevron-right</v-icon>
      </template>
      <template v-slot:item="{ item }">
        <v-breadcrumbs-item style="margin-left:-10px;margin-right:-10px"
        >
      
          <v-avatar v-if="!item.isDir" >
            <v-img max-height="15px"  max-width="15px"  src="./../../public/file-icons/racket-logo.svg"/>
          </v-avatar> <span class="white--text">{{ item.text }}</span>
        </v-breadcrumbs-item>
      </template>
    </v-breadcrumbs>
    <v-spacer></v-spacer>
       <v-btn @click="executeFile"
              icon
              color="green"
      >
        <v-icon color="green" style="margin-right: 0">mdi-play</v-icon>
      </v-btn>
      <v-btn
             icon
             color="red"
      >
        <v-icon @click="stopExecution" color="red" style="margin-right: 0">mdi-stop</v-icon>
      </v-btn>
    </v-system-bar>
    
    <Split style="background-color: #1f2020;height: 97vh">
      <SplitArea  :size="15">
        <file-explorer style="overflow: scroll;height: 100vh"/>
      </SplitArea>
      <SplitArea :size="85">
        <Split direction="vertical" >
          <SplitArea :size="60">
            <MonacoEditor
                    
                    v-model="$store.state.file.content"
                    class="editor"
                    theme="vs-dark"
                    language="scheme"
                    :options="options"
                    @change="onChange"
                    height="100%"
                    :editorBeforeMount="registerLang"
                    :editorMounted="onMount"
            ></MonacoEditor>
          </SplitArea>
          <SplitArea :size="40">
            <div ref="term" ></div>
          </SplitArea>
        </Split>
        
      </SplitArea>
    </Split>
    
  </v-app>
</template>

<script>
// @ is an alias to /src
import MonacoEditor from 'monaco-editor-vue';
const { spawn } = require("child_process");
const { ipcRenderer} = require('electron')
import FileExplorer from "../components/fileExplorer";
import { Terminal } from 'xterm';
import LocalEchoController from 'local-echo';
import { FitAddon } from 'xterm-addon-fit';
const fs = require('fs');
const path = require('path');

export default {
  name: 'Home',
  components: {
    FileExplorer,
    MonacoEditor
  },
  mounted() {
    this.term = new Terminal({
      convertEol:true,
      theme:{
        background:"#1f2020",
        brightCyan:"#13b9da"
      }
    });
    this.term.open(this.$refs.term);
    this.localEcho = new LocalEchoController(null,{
      isIncompleteInput:function (input) {
        if (input==='') return true;
        const count = (input.match(/"/g)||[]).length
        console.log(count % 2===0);
        return count % 2===1
      }
    });
    let localEcho = this.localEcho;
    this.fitAddon = new FitAddon();
    this.term.loadAddon(localEcho);
    this.term.loadAddon(this.fitAddon);
    this.fitAddon.fit();
    //this.term.write('Hello from \x1B[1;3;31mxterm.js\x1B[0m $ ');
    // spawn the process from the get-go
    const vapp = this;
    ipcRenderer.on('terminal-output',function (event, args) {
      
      console.log(args);
      if (args.endsWith("> ")){
        /*localEcho.read("> ")
                .then(input => {
                  //alert(`User entered: ${input}`)
                  ipcRenderer.send('terminal-input',input)
                })
                .catch(error => alert(`Error reading: ${error}`));*/
      }
      else{
        vapp.term.write(args);
      }
    })
    
  },
data() {
  return {
    term:null,
    fitAddon:null,
    options: {
      //Monaco Editor Options
      //minimap:true,
      automaticLayout: true,
      matchBrackets:"always",
      //renderIndentGuides:true,
      dragAndDrop:true,
      suggest: {
        showWords:true
      },
      suggestOnTriggerCharacters:true,
      parameterHints:{
        cycle:true,
        enabled:true
      },
      scrollBeyondLastLine:false,
      statusBar:{
        visible:true
      }
      
    },
    code:";code",
    selectedItem:null,
    outputText:""
  }
},
methods: {
  onChange(value) {
    //console.log(value);
    //send the value to the file
    fs.writeFileSync(this.$store.state.file.path,value)
    
  },
  show(value){
    //console.log(value)
    //this.code = value
    //console.log(JSON.parse(JSON.stringify(value)))
    this.code = value
  },
  registerLang(monaco){
    //console.log(monaco)
    monaco.languages.register({id:'scheme'})
    monaco.languages.registerCompletionItemProvider('scheme', {
      provideCompletionItems: () => {
        var suggestions = [{
          label: 'simpleText',
          kind: monaco.languages.CompletionItemKind.Text,
          insertText: 'simpleText'
        }, {
          label: 'testing',
          kind: monaco.languages.CompletionItemKind.Keyword,
          insertText: 'testing(${1:condition})',
          insertTextRules: monaco.languages.CompletionItemInsertTextRule.InsertAsSnippet
        }, {
          label: 'if-then-else',
          kind: monaco.languages.CompletionItemKind.Snippet,
          insertText: [
            'if (${1:condition}) {',
            '\t$0',
            '} else {',
            '\t',
            '}'
          ].join('\n'),
          insertTextRules: monaco.languages.CompletionItemInsertTextRule.InsertAsSnippet,
          documentation: 'If-Else Statement'
        }];
        return { suggestions: suggestions };
      }
    });
    
    
    
  },
  executeFile(){
    this.stopExecution();
    this.outputText = "";
    const path = this.$store.state.file.path
    if (!path) return false;
    const ls = spawn(`racket`,['-i',`-e`,`${this.$store.state.file.content}`]);
    const vapp = this;
    this.term.write('\x1bc')
    //this.term.clear()
    ls.stdout.on("data", data => {
      //console.log(`stdout: ${data}`);
      vapp.outputText += data;
      
      if (data.toString().endsWith("> ")){
        vapp.term.write(`\x1B[1;36m${data.slice(0, data.length - 2)}\x1b[0m`)
        console.log("input")
        vapp.localEcho.read("> ")
                .then(input => {
                  
                  console.log(`User entered: ${input}`)
                  //input =  input.split('\n').join("\\n");
                  console.log("writing")
                  if (!['(','"',','].includes( input.split('')[0])){
                    if ( input.split('')[0]==="'") input = `"${input}"`;
                    input = `(display ${input.split(" ")[0].split('\n')[0]})`
                    //console.log(input)
                  }
                  ls.stdin._write(input,'utf-8',(a)=>{console.log(`Something happened ${a}  `)})
                })
                .catch(error => alert(`Error reading: ${error}`));
      }
    else {
      console.log(JSON.stringify(data.toString()));
      console.log(data)
        vapp.term.writeln(`\x1B[1;36m${data}\x1b[0m`)
        
      }
      //vapp.term.write('\n')
    });
  
    ls.stderr.on("data", data => {
      console.log(`stderr: ${data}`);
      vapp.term.write(`\x1B[1;3;31m${data}\x1B[0m`)
      //vapp.outputText += data
    });
  
    ls.on('error', (error) => {
      console.log(`error: ${error.message}`);
      //vapp.outputText += error.message;
    });
  
    ls.on("close", code => {
      console.log(`child process exited with code ${code}`);
    });
    // racket -i -e "$(cat test.rkt)(display \"\n\")" opens a REPL. We need to pipe this somehow into the bottom editor.
    
  },
  stopExecution(){
    ipcRenderer.send('stop-process')
  },
  onMount(monaco,editor){
   // console.log(editor)
  }
},
  computed:{
    pathBreakdown(){
      return (this.$store.state.file.path||"").split("/").map((item,index,array)=> {
        //console.log(index===array.length-1)
        return {text: item,
          isDir:index!==array.length-1}
      }).splice(1)
      
      
    }
  }
}
</script>
<style lang="scss">
  .editor{
  
  }
  .gutter {
    background-color: #474747 !important;
  }
  .titlebar {
    -webkit-user-select: none;
    -webkit-app-region: drag;
  }
  .home{
    height: 100%;
    background-color: #1f2020;
  }
</style>


<style scoped>
  @import '~xterm/css/xterm.css';
</style>
