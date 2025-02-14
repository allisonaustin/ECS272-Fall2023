<script lang="ts">
import * as d3 from "d3";
import * as d3Sankey from 'd3-sankey';
import axios from 'axios';
import { isEmpty, debounce } from 'lodash';

import { Graph, ComponentSize, Margin } from '../types';

// music & mental health survey data
// https://www.kaggle.com/datasets/catherinerasgaitis/mxmh-survey-results
const data = await d3.csv('../../data/mxmh_survey_results.csv');
const genreGrouped = groupBy(data, "Fav genre")
const workingGrouped = groupBy(data, "While working")
const anxietyGrouped = groupBy(data, "Anxiety")
const depressionGrouped = groupBy(data, "Depression")
const musicEffectGrouped = groupBy(data, "Music effects")

let sankeyData = {"nodes": [], "links": []}

// nodes 
sankeyData.nodes.push({ "name": 'Listen while working? Yes' })
sankeyData.nodes.push({ "name": 'Listen while working? No' })
sankeyData.nodes.push({ "name": 'Anxiety Level 0-3' })
sankeyData.nodes.push({ "name": 'Anxiety Level 4-7' })
sankeyData.nodes.push({ "name": 'Anxiety Level 8-10' })
sankeyData.nodes.push({ "name": "Depression Level 0-3" })
sankeyData.nodes.push({ "name": "Depression Level 4-7"})
sankeyData.nodes.push({ "name": "Depression Level 8-10"})
sankeyData.nodes.push({ "name": 'Improved mental health' })
sankeyData.nodes.push({ "name": 'Worsened mental health' })
sankeyData.nodes.push({ "name": 'No effect on mental health' })


// genre -> listen while working
let genreWorking = []
Object.keys(genreGrouped).forEach(g => {
if (g != "") {
    sankeyData.nodes.push({ "name": g })
    genreWorking = (genreGrouped[g].map(function (obj) {
        return obj['While working']
    }))

    // getting counts of each genre for working/not working
    let w_counts = {}
    genreWorking.forEach(g => {
        if (w_counts[g] == null) w_counts[g] = 0
        if (g == 'Yes')
            w_counts[g]++
        else if (g == 'No') {
            w_counts[g]++
        }
    })
    sankeyData.links.push({ "source": g, "target": 'Listen while working? Yes', "value": isNaN(w_counts['Yes']) ? 0 : +w_counts['Yes'] })
    sankeyData.links.push({ "source": g, "target": 'Listen while working? No', "value": isNaN(w_counts['No']) ? 0 : +w_counts['No'] })
    }
})

// while working -> anxiety
let anxiety = [];
Object.keys(workingGrouped).forEach(g => {
if (g != '') {
    anxiety = (workingGrouped[g].map(function (obj) {
        return obj['Anxiety']
    }))

    let a_counts = {}
    a_counts['0-3'] = 0
    a_counts['4-7'] = 0
    a_counts['8-10'] = 0
    anxiety.forEach(a => {
        if (a <=3 && a >= 0) {
            a_counts['0-3'] += 1;
        } else if (a <=7 && a > 3) {
            a_counts['4-7'] += 1;
        } else if (a <= 10 && a > 7) {
        a_counts['8-10'] += 1;
        }
    });
    sankeyData.links.push({ "source": 'Listen while working? ' + g, "target": 'Anxiety Level 0-3', "value": +a_counts['0-3'] })
    sankeyData.links.push({ "source": 'Listen while working? ' + g, "target": 'Anxiety Level 4-7', "value": +a_counts['4-7'] })
    sankeyData.links.push({ "source": 'Listen while working? ' + g, "target": 'Anxiety Level 8-10', "value": +a_counts['8-10'] })
}
})

// anxiety -> depression level
let depression = [];
Object.keys(anxietyGrouped).forEach(g => {
depression = (anxietyGrouped[g].map(function(obj) {
    return obj['Depression']
}))

let d_counts = {}
d_counts['0-3'] = 0
d_counts['4-7'] = 0
d_counts['8-10'] = 0
depression.forEach(d => {
      if (d <=3 && d >= 0) {
            d_counts['0-3'] += 1;
        } else if (d <=7 && d > 3) {
            d_counts['4-7'] += 1;
        } else if (d <= 10 && d > 7) {
          d_counts['8-10'] += 1;
        }
    });
    sankeyData.links.push({ "source": 'Anxiety Level ' + levels(g), "target": 'Depression Level 0-3', "value": +d_counts['0-3'] })
    sankeyData.links.push({ "source": 'Anxiety Level ' + levels(g), "target": 'Depression Level 4-7', "value": +d_counts['4-7'] })
    sankeyData.links.push({ "source": 'Anxiety Level ' + levels(g), "target": 'Depression Level 8-10', "value": +d_counts['8-10'] })
  })

  
  // depression level -> music helping ?
  let musicEffect = []
  Object.keys(depressionGrouped).forEach(g => {
      if (g != "") {
        musicEffect = (depressionGrouped[g].map(function(obj) {
          return obj['Music effects']
        }))

        let e_counts = {}
        musicEffect.forEach(e => {
            if (e_counts[e] == null) e_counts[e] = 0
            if (e == 'Improve') {
              e_counts[e] += 1
            }
            else if (e == 'Worsen') {
              e_counts[e] += 1
            } else if (e == 'No effect') {
              e_counts[e] += 1
            }
        })
      sankeyData.links.push({ "source": 'Depression Level '+ levels(g), "target": 'Improved mental health', "value": isNaN(e_counts['Improve']) ? 0 : +e_counts['Improve'] })
      sankeyData.links.push({ "source": 'Depression Level '+ levels(g), "target": 'Worsened mental health', "value": isNaN(e_counts['Worsen']) ? 0 : +e_counts['Worsen'] })
      sankeyData.links.push({ "source": 'Depression Level '+ levels(g), "target": 'No effect on mental health', "value": isNaN(e_counts['No effect']) ? 0 : +e_counts['No effect'] })
      }
  })


function groupBy(arr, property) {
    return arr.reduce(function (acc, obj) {
        let key = obj[property]
        if (!acc[key]) {
            acc[key] = []
        }
        acc[key].push(obj)
            return acc
        }, 
    {})
}

function levels(value) {
    if (parseInt(value) <= 3) return '0-3'
    if (parseInt(value) > 3 && value < 8) return '4-7'
    if (parseInt(value) <= 10 && value >= 8) return '8-10'
}

export default {
    data() {
        return {
            nodes: [] as Graph[],
            size: { width: 900, height: 300 } as ComponentSize,
            margin: {left: 40, right: 40, top: 15, bottom: 40} as Margin
        }
    },
    computed: {
        rerender() {
            return (!isEmpty(this.nodes)) && this.size
        }
    },
    created() { 
        if (isEmpty(data)) return;
        this.nodes = sankeyData;
    },
    methods: {
        onResize() {
            let target = this.$refs.sankeyContainer as HTMLElement
            if (target === undefined) return;
            this.size = { width: target.clientWidth, height: target.clientHeight };
        },
        initChart() {
            // references: 
            // https://github.com/d3/d3-sankey
            // https://observablehq.com/@d3/sankey-component
            let chartContainer = d3.select('#sankey-svg')
                .attr('viewBox', [0, 0, this.size.width + 10, this.size.height])
                .attr('style', 'max-width: 100%; height: auto;')


            const format = d3.format(",.0f")

            // Constructs and configures a Sankey generator.
            let sankey = d3Sankey.sankey()
                .nodeId(d => d.name)
                .nodeAlign(d3Sankey.sankeyLeft) // d3.sankeyLeft, etc.
                .nodeWidth(15)
                .nodePadding(10)
                .extent([[1, 5], [this.size.width - 1, this.size.height - 5]]);
        
            
            const {nodes, links} = sankey({
                nodes: sankeyData.nodes.map(d => Object.assign({}, d)),
                links: sankeyData.links.map(d => Object.assign({}, d))
            });

            const colors = d3.scaleOrdinal(d3.schemeTableau10);

            // nodes
            const node = chartContainer.append('g')
                .attr('stroke', '#000')
                .selectAll('rect')
                .data(nodes)
                .join('rect')
                    .attr('x', (d) => d.x0)
                    .attr('y', (d) => d.y0)
                    .attr('height', (d) => d.y1 - d.y0)
                    .attr('width', (d) => d.x1 - d.x0)
                    .attr('fill', d => colors(d.name))
                    .style('opacity', '0.5') 
                    .attr('class', 'node')  
                .on('mouseover', (event, v) => {
                    const linkedNodes = [];
                    var traverse = [{
                        linkType : 'sourceLinks',
                        nodeType : 'target',
                        }, {
                        linkType : 'targetLinks',
                        nodeType : 'source',
                    }];

                    traverse.forEach((step) => {
                        v[step.linkType].forEach((l) => {
                            linkedNodes.push(l[step.nodeType]);
                        });
                    });
                    
                    // Update linked nodes style
                    d3.selectAll('.node').style('opacity', r => linkedNodes.find(remainingNode => remainingNode.name === r.name) ? '1' : '0.5');
                    
                    // Update hovered node style
                    d3.select(event.currentTarget).style('opacity', '1');
                    
                    // Update links style
                    d3.selectAll('.link').style('stroke', p => (p && (p.source.name === v.name || p.target.name === v.name)) ? colors(v.name) : '#aaa')

                    d3.selectAll('.link').style('opacity', p => (p && (p.source.name === v.name || p.target.name === v.name)) ? '0.5' : '0.3');
                })
                .on('mouseout', (event, v) => {
                    // Update nodes style
                    d3.selectAll('.node').style('opacity', '0.5');
                    
                    // Update links style
                    d3.selectAll('.link').style('opacity', '0.3');
                    d3.selectAll('.link').style('stroke', '#aaa')
                })
            
            
            // node titles
            node.append("title")
                .text(d => `${d.name}\n${format(d.value)}`);

            // links
            const link = chartContainer.append('g')
                .selectAll()
                .data(links)
                .join('g')

            link.append('path')
                .attr('d', d3Sankey.sankeyLinkHorizontal())
                .transition()
                .duration(350) 
                .attr('stroke', (d) => d.uid)
                .attr('stroke-width', d => Math.max(1, d.width))
                .attr('opacity', 0.3)
                .attr('stroke', '#aaa')
                .attr('fill', 'none')
                .attr('class', 'link')


            link.append('title')
                .text(d => `${d.source.name} -> ${d.target.name}\n${format(d.value)}`)

            // Adding labels to nodes
            const labels = chartContainer.append('g')
                .selectAll()
                .data(nodes)
                .join('text')
                    .attr('x', d => d.x0 < this.size.width / 2 ? d.x1 + 6 : d.x0 - 6)
                    .attr('y', d => (d.y1 + d.y0) / 2)
                    .attr('dy', '0.35em')
                    .attr('text-anchor', d => d.x0 < this.size.width / 2 ? 'start' : 'end')
                    .text(d => d.name)
                    .style('font-size', '10')
                    .style('font-family', 'monospace')
            
            // chart title
            const title = chartContainer.append('g')
                .append('text')
                .attr('transform', `translate(${this.size.width / 2}, ${this.size.height - this.margin.bottom - 40})`)
                .attr('dy', '0.5rem') 
                .style('text-anchor', 'middle')
                .style('font-weight', 'bold')
                .style('font-family', 'monospace')
                .style('font-size', '18')
                .text('Context View of Survey Responses')
        }
    },
    watch: {
        rerender(newSize) {
            if (!isEmpty(newSize)) {
                d3.select('#sankey-svg').selectAll('*').remove()
                this.initChart()
            }
        }
    },
    mounted() {
        window.addEventListener('resize', debounce(this.onResize, 100)) 
        this.onResize()
    },
    beforeDestroy() {
       window.removeEventListener('resize', this.onResize)
    }
}

</script>
<template>
    <div class="chart-container d-flex" ref="secondaryContainer">
        <svg id="sankey-svg" width="100%" height="100%">
        </svg>
    </div>
</template>

<style scoped>
.chart-container{
    height: 100%;
}
</style>

