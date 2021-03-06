<template lang="pug">
.simulation-statistics.scrollbars
  .columns
    .column.is-one-half
    .column
      b-field.generation-controls
        p.control
          b-tooltip(label="Run the simulation again", position="is-left")
            b-button.btn-dark(@click="restart()", :loading="isContinuing", icon-right="reload")
        p.control
          b-tooltip(label="Run the simulation with a new random seed", position="is-left")
            b-button.btn-dark(@click="shuffleSimulation()", :loading="isContinuing", icon-right="dice-5")
        p.control
          b-tooltip(label="See more generations", position="is-left")
            b-button.btn-dark(@click="keepGoing()", :loading="isContinuing", icon-right="layers-plus")
  .field.is-grouped.is-grouped.multiline
    .tags
      .tag.is-large.clickable(v-for="plot in hiddenPlots", :style="{ color: plot.titleColor }", @click="showPlot(plot)")
        b-icon(icon="plus-box")
        span {{ plot.label | startCase }}

  .columns.is-multiline
    .column.is-half-tablet.is-one-third-fullhd(v-for="plot in plots", :key="plot.label")
      h2.heading.plot-title.clickable(:style="{ color: plot.titleColor }", @click="hidePlot(plot)")
        b-icon(icon="minus-box")
        | &nbsp;
        span.name {{ plot.label | startCase }}
      TraitPlot(:data="plot.data", :label="plot.label | startCase", :color="plot.color", @click="loadGeneration")
  .columns
    .column
      h2.heading.plot-title
        span.name Overview
      OverviewPlot(:data="plots", label="Overview")
</template>

<script>
import { mapGetters, mapActions } from 'vuex'
import _mapValues from 'lodash/mapValues'
import _findIndex from 'lodash/findIndex'
import chroma from 'chroma-js'
import traitColors from '@/config/trait-colors'
import TraitPlot from '@/components/trait-plot'
import OverviewPlot from '@/components/overview-plot'

const titleColors = _mapValues(traitColors, c => chroma(c).desaturate(1).css())

export default {
  name: 'SimulationStatistics'
  , props: {
  }
  , data: () => ({
    traitColors
    , populationColor: traitColors.population
    , titleColors
    , hiddenPlots: []
  })
  , components: {
    TraitPlot
    , OverviewPlot
  }
  , created(){
  }
  , deactivated(){
  }
  , beforeDestroy(){
  }
  , mounted(){
  }
  , computed: {
    ...mapGetters('simulation', {
      generationIndex: 'currentGenerationIndex'
      , statistics: 'statistics'
      , traits: 'traits'
      , isLoading: 'isLoading'
      , isContinuing: 'isContinuing'
    })
    , plots(){
      return [{
        color: traitColors.population
        , titleColor: traitColors.population
        , data: this.populationData
        , label: 'population'
        , yAxisID: 'population'
      }].concat(
        this.traits.map(t => ({
          color: traitColors[t]
          , titleColor: titleColors[t]
          , data: this.traitData[t]
          , label: t
        }))
      ).filter(v => _findIndex(this.hiddenPlots, ['label', v.label]) < 0)
    }
    , traitData(){
      if (!this.statistics){ return {} }
      let data = {}
      this.traits.forEach(t => {
        data[t] = this.statistics.generation_statistics.map(d => d[t])
      })
      return data
    }
    , populationData(){
      if (!this.statistics){ return [] }
      return this.statistics.generation_statistics.map(d => d.population)
    }
  }
  , watch: {
  }
  , methods: {
    showPlot(plot){
      let index = this.hiddenPlots.indexOf(plot)
      this.hiddenPlots.splice(index, 1)
    }
    , hidePlot(plot){
      this.hiddenPlots.push(plot)
    }
    , shuffleSimulation(){
      let seed = (Math.random() * 1000) | 0
      this.$store.dispatch('simulation/setConfig', { seed })
      this.restart()
    }
    , loadGeneration(generationIndex){
      let route = this.$route
      let params = route ? route.params : {}
      let query = route ? route.query : {}
      this.$router.push({ name: 'viewscreen', params: { ...params, generationIndex }, query })
    }
    , ...mapActions('simulation', {
      restart: 'run'
      , keepGoing: 'continue'
    })
  }
}
</script>

<style lang="sass" scoped>
.simulation-statistics
  padding: 6em 2em 0
  overflow-y: scroll
.plot-title
  margin-top: 1em
  text-align: center
  white-space: nowrap
  .icon
    position: relative
    top: 2px
  .name
    font-size: 24px
.field.generation-controls
  justify-content: flex-end
</style>
