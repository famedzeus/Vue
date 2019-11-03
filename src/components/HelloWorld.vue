<template>
  <div class="hello">
    {{MIDIDataIn}}
    <span v-for="(tick,index) in sequence" :style="{'background-color':Math.floor(measure)==index+1?'red':'white'}">
      :{{index+1}}:
    </span><br/>
    <span v-for="(tick,index) in sequence" :style="{'background-color':Math.floor(measure)==index+1?'red':'white'}">
      :{{tick.notes.length}}:
    </span>
    Measure:{{measure}}

    <div v-on:click="start()">Start</div>
    <div v-on:click="stop()">Stop</div>

    <!-- <div v-on:click="addNoteOn(244,100,255)">add note</div> -->


  </div>
  

</template>

<script>
import moment from "moment";
import VDatePicker from "vue-date-time-picker/src/date_picker.vue";

export default {
  data () {
      return {
          myDate: '',
          MIDIDataIn:[],
          outputs:[],
          sequence:[],
          measure:1,
          looper:null
      };
  },
  name: "HelloWorld",
  components: { VDatePicker },
  props: {
    msg: String
  },
  mounted: function() {
      navigator.requestMIDIAccess()
          .then(this.onMIDISuccess, this.onMIDIFailure);
      for(let i = 0; i < 96 ;i++){
        this.sequence.push({notes:[],control:[]})
      }
  },
  methods: {
    onMIDISuccess(midiAccess) {
        let vm = this
        for (var input of midiAccess.inputs.values()) {
            input.onmidimessage = this.processMIDIMessage;
        }
        this.outputs = []
        var iter = midiAccess.outputs.values();
        for (var i = iter.next(); i && !i.done; i = iter.next()) {
          this.outputs.push(i.value);
        }
    },
    onMIDIFailure() {
        console.log('Could not access your MIDI devices.');
    },
    processMIDIMessage(message) {
      let vm= this
      var command = message.data[0];
      var note = message.data[1];
      var velocity = (message.data.length > 2) ? message.data[2] : 0; // a velocity value might not be included with a noteOff command

      // clock coming in
      if(message.data[0]!==248 && message.data[0]!==254){
        console.log(message.data[0] + ":"+ message.data[1])
      }
      
      switch (command) {
          case 251:
            // Start
            this.start()
            break;
          case 252:
            //Stop
            this.stop()
            break;
          case 176:
            if(message.data[1]==64){
              console.log("Foot sustain")
            }
            break;
          case 144: // noteOn
              if (velocity > 0) {
                  this.addNoteOn(command, note, velocity);
              } else {
                  this.addNoteOff(command, note);
              }
              break;
          case 128: // noteOff
              this.addNoteOff(command,note);
              break;
          // we could easily expand this switch statement to cover other types of commands such as controllers or sysex
      }
    }, 
    start(){
      let vm = this
      this.measure = 1
      // Clock out
      clearInterval(this.looper)
      this.looper = setInterval(
          function() {
            if(vm.measure == 96){
              vm.measure = 1
            }
            vm.outputs[0].send([248]);
            vm.sequence[vm.measure-1].notes.forEach(note => {
              vm.outputs[0].send([note.command,note.note,note.velocity])
            })
            vm.measure+=1;
          },
          25  
      );    
    },
    stop(){
      clearInterval(this.looper)
      this.measure = 1
    },
    addNoteOn(command,note,velocity){
      if(this.looper){
        this.sequence[this.measure-1].notes.push({command:command,note:note,velocity:velocity})
      }
      console.log("Note on:"+note)
    },
    addNoteOff(command,note){
      if(this.looper){
        this.sequence[this.measure-1].notes.push({command:command,note:note,velocity:0})
      }
      console.log("note off:"+note)
    }
  }
};
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
h3 {
  margin: 40px 0 0;
}
ul {
  list-style-type: none;
  padding: 0;
}
li {
  display: inline-block;
  margin: 0 10px;
}
a {
  color: #42b983;
}
</style>
