<template lang="pug">
.viewer
  transition(name="slide-down", appear, @after-enter="fixLayout", @after-leave="fixLayout")
    .top-controls(v-show="!showConfig && !showIntro")
      FloatingPanel(size="is-medium", :close-on-click="false")
        template(#activator)
          b-icon.icon-btn(icon="layers-search", size="is-medium")
        .item
          b-tooltip(label="Toggle sight range indicators", position="is-left")
            b-icon.icon-btn(icon="eye", size="is-medium", :class="{ active: showSightIndicator }", @click.native.stop="showSightIndicator = !showSightIndicator")
        .item
          b-tooltip(label="Color blobs based on speed (brown: slow, red: fast)", position="is-left")
            b-icon.icon-btn(icon="run-fast", size="is-medium", :class="{ active: showSpeedIndicator }", @click.native.stop="showSpeedIndicator = !showSpeedIndicator")
        .item
          b-tooltip(label="Toggle energy indicators", position="is-left")
            b-icon.icon-btn(icon="battery-charging", size="is-medium", :class="{ active: showEnergyIndicator }", @click.native.stop="showEnergyIndicator = !showEnergyIndicator")
        .item
          b-tooltip(label="Toggle food indicators", position="is-left")
            b-icon.icon-btn(icon="silverware-fork-knife", size="is-medium", :class="{ active: showFoodIndicator }", @click.native.stop="showFoodIndicator = !showFoodIndicator")

      b-icon.icon-btn(icon="cctv", :class="{ active: followCreature }", size="is-medium", @click.native.stop="followCreature = !followCreature")

      .under
        CreatureFlowerChart(v-if="followCreature && followCreatureId", :id="followCreatureId")

  .screen
    b-loading.loading-cover(:is-full-page="false", :active="!showIntro && isLoading")
    GenerationViewer(
      ref="generationViewer"
      , :generation-index="genIndex"
      , :step-time="stepTime"
      , :sight-indicators="showSightIndicator"
      , :speed-indicators="showSpeedIndicator"
      , :energy-indicators="showEnergyIndicator"
      , :food-indicators="showFoodIndicator"
      , :followCreatureId="followCreature ? followCreatureId : undefined"
      , @tap-creature="followCreatureId = $event.creature.id; followCreature = true"
    )
  transition(name="fade", appear)
    .controls(v-if="!showIntro", v-show="!showConfig")
      .inner
        b-field.extras(grouped, position="is-centered", :class="{ active: playthrough }")
          b-field
            b-tooltip(:label="'Playthrough ' + (playthrough ? '(is on)' : '(is off)')", position="is-top")
              b-icon.icon-btn(title="play through", size="is-medium", icon="transfer-right", @click.native="playthrough = !playthrough")
        b-field(grouped, position="is-centered")
          b-field
            b-icon.icon-btn(icon="chevron-left", size="is-large", @click.native="prevGeneration()")
          b-field
            b-icon.icon-btn(:icon="paused ? 'play' : 'pause'", size="is-large", @click.native="togglePlay()")
          b-field
            b-icon.icon-btn(icon="chevron-right", size="is-large", @click.native="nextGeneration()")

        .right
          FloatingPanel(size="is-medium", direction="up", :close-on-click="false")
            template(#activator)
              b-icon.icon-btn(icon="clock-fast", size="is-medium")
            .item
              vue-slider(
                v-model="playbackSpeed"
                , :min="1"
                , :max="13"
                , :interval="1"
                , :height="200"
                , direction="btt"
                , :contained="true"
              )
      AudioScrubber(:progress="progress", @scrub="onScrub", @scrubstart="onScrubStart", @scrubend="onScrubEnd")
</template>

<script>
import { mapGetters } from 'vuex'
import _findIndex from 'lodash/findIndex'
import Copilot from 'copilot'
import AudioScrubber from '@/components/audio-scrubber'
import GenerationViewer from '@/components/generation-viewer'
import FloatingPanel from '@/components/floating-panel'
import CreatureFlowerChart from '@/components/creature-flower-chart'

export default {
  name: 'ViewScreen'
  , props: {
    showConfig: Boolean
    , showIntro: Number
  }
  , provide(){
    const self = this
    return {
      getTime(){ return self.player.time }
      , getStep(){ return self.player.time / self.stepTime }
    }
  }
  , data: () => ({
    paused: true
    , playthrough: false
    , progress: 0
    , showSightIndicator: false
    , showSpeedIndicator: false
    , showEnergyIndicator: false
    , showFoodIndicator: false
    , followCreature: false
    , followCreatureId: 0
  })
  , components: {
    AudioScrubber
    , GenerationViewer
    , FloatingPanel
    , CreatureFlowerChart
  }
  , created(){
    this.player = Copilot.Player({ totalTime: 1 })
  }
  , deactivated(){
    this.player.togglePause(true)
  }
  , beforeDestroy(){
    this.player.togglePause(true)
    this.player.destroy()
  }
  , mounted(){
    this.player.on('update', () => {
      this.progress = this.player.time / this.totalTime * 100
    })
    this.player.on('togglePause', () => {
      if ( this.player.paused !== this.paused ){
        this.paused = this.player.paused
      }
    })
    this.player.on('end', () => {
      if ( this.playthrough ){
        this.nextGeneration()
      }
    })
  }
  , computed: {
    stepTime(){
      return 100 //Math.pow(10, 1.5 * (1 - this.playbackSpeed/5)) * 100
    }
    , playbackSpeed: {
      get(){
        let s = this.player.playbackRate
        return Math.log2(s) * 2 + 7
      }
      , set(s){
        this.player.playbackRate = Math.pow(2, (s - 7)/2)
      }
    }
    , totalTime(){
      if ( !this.generation ){ return 1 }
      return this.stepTime * this.generation.steps
    }
    , generation(){
      return this.getCurrentGeneration()
    }
    , genIndex: {
      get(){
        return this.generationIndex
      }
      , set(v){
        this.loadGeneration(v)
      }
    }
    , tourStepNumber(){
      return this.$route.query.intro | 0
    }
    , ...mapGetters('simulation', {
      getCurrentGeneration: 'getCurrentGeneration'
      , generationIndex: 'currentGenerationIndex'
      , stats: 'statistics'
      , isLoading: 'isLoading'
    })
  }
  , watch: {
    totalTime(){
      let p = this.player.time / this.player.totalTime
      this.player.totalTime = this.totalTime
      this.player.time = this.totalTime * p
    }
    , tourStepNumber(step, oldStep){
      if (step >= 2){
        this.playthrough = true
        this.playbackSpeed = 2
        this.paused = false
      } else {
        this.playbackSpeed = 5
        this.paused = true
        if (oldStep > 1 && !step){
          this.playthrough = true
          this.genIndex = 0
          this.paused = false
        }
      }

      if (step === 2){
        this.genIndex = 0
      }

      if (step === 3){
        this.genIndex = 0
        this.followCreatureId = this.generation.creatures[2].id
      }

      if (step === 4){
        this.genIndex = 0
        this.showEnergyIndicator = true
      } else {
        this.showEnergyIndicator = false
      }

      this.followCreature = step === 3
    }
    , paused(){
      this.player.togglePause(this.paused)
    }
    , genIndex(g, oldg){
      if (g === oldg){ return }
      let index = _findIndex(this.generation.creatures, c => c.id === this.followCreatureId)

      if ( index < 0 ){
        this.followCreature = false
        this.followCreatureId = null
      }

      if (this._inactive) return

      this.paused = true
      if ( this.player ){
        this.player.seek(0)
      }
      if (!this.playthrough) return
      setTimeout(() => {
        this.paused = false
      }, 500)
    }
    , followCreature(f){
      if ( f && !this.followCreatureId ){
        this.followCreatureId = this.generation.creatures[0].id
      }
    }
  }
  , methods: {
    togglePlay(){
      this.paused = !this.paused
    }
    , updateTime(time){
      if ( time !== this.player.time ){
        this.player.seek(time)
      }
    }
    , onScrub(progress){
      this.updateTime(progress * this.totalTime / 100)
    }
    , onScrubStart(){
      this.wasPaused = this.paused
      this.paused = true
    }
    , onScrubEnd(){
      this.paused = this.wasPaused
    }
    , loadGeneration(v){
      this.$store.dispatch('simulation/loadGeneration', v)
    }
    , prevGeneration(){
      if ( !this.generation ){ return }
      if ( this.genIndex <= 0 ){ return }
      this.genIndex -= 1
    }
    , nextGeneration(){
      if ( !this.generation ){ return }
      if ( this.genIndex >= this.stats.num_generations - 1 ){ return }
      this.genIndex += 1
    }
    , fixLayout(){
      let viewer = this.$refs.generationViewer
      if ( viewer ){
        viewer.onResize()
      }
    }
  }
}
</script>

<style lang="sass" scoped>
.viewer
  position: relative
  display: flex
  flex-direction: column
  .screen
    flex-grow: 1
    overflow: hidden
    display: flex
    align-items: stretch
    > *
      flex: 1

.top-controls
  position: absolute
  top: 1.5rem
  right: 1.5rem
  z-index: 1
  text-align: right

  > *
    &:not(:first-child)
      margin-left: 1rem

  @media screen and (max-width: $tablet)
    top: 5rem

    .under
      transform: scale(0.6)
      transform-origin: top right

.loading-cover
  z-index: 1
.controls
  position: absolute
  bottom: 0
  left: 0
  right: 0
  align-items: baseline
  pointer-events: none

  >>> .icon,
  .right,
  >>> .scrubber
    pointer-events: all

  >>> .vue-slider-process
    background-color: $primary
  >>> .vue-slider-rail
    background-color: $sand
  >>> .vue-slider-dot-tooltip-inner
    background-color: $primary

  .left
    position: absolute
    left: 1.5rem
    bottom: 1.5rem
    z-index: 1
  .right
    position: absolute
    right: 1.5rem
    bottom: 1.5rem

  >>> .field
    margin-bottom: 0.5em

  .extras
    margin-bottom: 0

>>> .scrubber .inner
  background: $sand
</style>
