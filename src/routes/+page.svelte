<script lang="ts">
    import TopAppBar, { Row, Section, Title as TopAppBarTitle, AutoAdjust } from '@smui/top-app-bar';
    import Button, { Label as ButtonLabel, Icon as ButtonIcon } from '@smui/button';
    import Dialog, { Title as DialogTitle, Content as DialogContent } from '@smui/dialog';
    import LinearProgress from '@smui/linear-progress';
    import LayoutGrid, { Cell } from '@smui/layout-grid';
    import Paper, { Title as PaperTitle, Subtitle as PaperSubtitle, Content as PaperContent } from '@smui/paper';
    import FormField from '@smui/form-field';
    import Radio from '@smui/radio';
    import Slider from '@smui/slider';

    import { onMount } from 'svelte';
    import * as d3 from 'd3';

    let topAppBar = $state(null);

    let loading = $state(true);

    function sleep(time) {
        return new Promise((resolve) => setTimeout(resolve, time));
    }

    // Timespan Selection
    const granularityOptions = $state([
        {
            name: "years",
        },
        {
            name: "months",
        },
        {
            name: "days",
        },
    ]);
    let granularity = $state("days");

    let minRange = $derived((granularity == "days" ? 1 : 0));

    const absoluteMinDate = new Date(2024, 0, 1);
    const absoluteMaxDate = d3.timeSecond.offset(new Date(2025, 1, 1), -1);

    const minDateIndex = 0;
    const maxDateIndex = d3.timeDay.count(absoluteMinDate, absoluteMaxDate);

    let sliderStartDateIndex = $state(minDateIndex);
    let sliderEndDateIndex = $state(maxDateIndex);

    let selectedStartDateIndex = $derived(sliderStartDateIndex);
    let selectedEndDateIndex = $derived(sliderEndDateIndex);

    function resetTimespan() {
        selectedStartDateIndex = minDateIndex;
        selectedEndDateIndex = maxDateIndex;

        sliderStartDateIndex = minDateIndex;
        sliderEndDateIndex = maxDateIndex;
    }

    let selectedStartDate = $derived(d3.timeDay.offset(absoluteMinDate, selectedStartDateIndex));
    let selectedEndDate = $derived(d3.timeSecond.offset(d3.timeDay.offset(absoluteMinDate, selectedEndDateIndex), -1));

    let selectedTimespanLength = $derived.by(
        () => {
            switch (granularity) {
                case "years":
                    return d3.timeYear.count(selectedStartDate, selectedEndDate) + 1;

                case "months":
                    return d3.timeMonth.count(selectedStartDate, selectedEndDate) + 1;

                case "days":
                    return d3.timeDay.count(selectedStartDate, selectedEndDate) + 1;
            }
        }
    );

    let previousColor = null;
    let selectedMatch = $state(null);

    let parseDate = d3.utcParse("%Y-%m-%d %H:%M:%S");
    let formatDate = d3.timeFormat("%d %B %Y");

    // BEGIN DATA PROCESSING FUNCTIONS

    let data = $state([]);

    let matches = $derived.by(
        () => {
            const result = [];

            const helper = {};

            for (let index = 0; index < data.length; index += 1) {
                if (!(data[index]["game_id"] in helper)) {
                    let match = {
                        id: data[index]["game_id"],
                        start: data[index]["game_start"],
                        duration: data[index]["game_duration"],
                        end: d3.timeSecond.offset(data[index]["game_start"], data[index]["game_duration"]),
                        mode: data[index]["game_mode"],
                        champions: [data[index]["champion_name"]],
                        win: ((data[index]["team_id"] == "100") && data[index]["win"] ? 1 : 2),
                        summoners: [
                            {
                                id: data[index]["summoner_id"],
                                name: data[index]["summoner_name"],
                                team: (data[index]["team_id"] == "100" ? 1 : 2),
                                position: data[index]["team_position"],
                                kills: data[index]["kills"],
                                deaths: data[index]["deaths"],
                                assists: data[index]["assists"],
                            }
                        ]
                    }

                    helper[data[index]["game_id"]] = match;

                    result.push(match);
                } else {
                    helper[data[index]["game_id"]].champions.push(data[index]["champion_name"]);
                    helper[data[index]["game_id"]].summoners.push(
                        {
                            id: data[index]["summoner_id"],
                            name: data[index]["summoner_name"],
                            team: (data[index]["team_id"] == "100" ? 1 : 2),
                            position: data[index]["team_position"],
                            kills: data[index]["kills"],
                            deaths: data[index]["deaths"],
                            assists: data[index]["assists"],
                        }
                    )
                }
            }

            return result;
        }
    );

    let selectedMatches = $derived(matches.filter((match) => (match.start >= selectedStartDate && match.start <= selectedEndDate)));

    function countMatches(matches, granularity, selectedStartDate, selectedTimespanLength) {
        const result = [];

        switch (granularity) {
            case "years":
                for (let i = 0; i < selectedTimespanLength; i += 1) {
                    result.push(0);
                }

                for (let match of matches) {
                    const index = d3.timeYear.count(selectedStartDate, match.start);
                    result[index] += 1;
                }

                break;

            case "months":
                for (let i = 0; i < selectedTimespanLength; i += 1) {
                    result.push(0);
                }

                for (let match of matches) {
                    const index = d3.timeMonth.count(selectedStartDate, match.start);
                    result[index] += 1;
                }

                break;

            case "days":
                for (let i = 0; i < selectedTimespanLength; i += 1) {
                    result.push(0);
                }

                for (let match of matches) {
                    const index = d3.timeDay.count(selectedStartDate, match.start);
                    result[index] += 1;
                }

                break;
        }

        return result;
    }

    function countGameModes(matches) {
        const result = [];

        for (let i = 0; i < 3; i += 1) {
            result[i] = 0;
        }

        for (let match of matches) {
            switch (match.mode) {
                case "CLASSIC":
                    result[0] += 1;
                    break;
                case "ARAM":
                    result[1] += 1;
                    break;
                case "SWIFTPLAY":
                    result[2] += 1;
                    break;
            }
        }

        return result;
    }

    // END DATA PROCESSING FUNCTIONS

    // BEGIN GRAPH DRAWING FUNCTIONS

    function drawMatchesOverTimeChart(chart, matchCounts, granularity) {
        chart.selectAll("*").remove();

        let xAxisHeight = 40;
        let yAxisWidth = 40;

        let chartWidth = chart.node().clientWidth - 2 * yAxisWidth;
        let chartHeight = chart.node().clientHeight - 2 * xAxisHeight;
        let chartX = 0 + yAxisWidth;
        let chartY = 0 + chartHeight + xAxisHeight;

        chart.append("text")
            .attr("x", chartX - 25)
            .attr("y", chartY - chartHeight - 20)
            .attr("fill", "white")
            .attr("font-size", "0.8em")
            .text("MATCH COUNT")

        chart.append("text")
            .attr("x", chartX + (chartWidth / 2))
            .attr("y", chartY + 40)
            .attr("fill", "white")
            .attr("font-size", "0.8em")
            .text("TIME")

        let barPaddedWidth = (chartWidth / matchCounts.length);

        let barPaddingX = 2;

        let barWidth = barPaddedWidth - barPaddingX;

        let maxBarHeight = chartHeight;

        let max = 0;

        for (let index = 0; index < matchCounts.length; index += 1) {
            let maxIndex = 0;
            max = matchCounts[maxIndex];

            for (let i = maxIndex; i < matchCounts.length; i += 1) {
                if (matchCounts[i] > max) {
                    maxIndex = i;
                    max = matchCounts[maxIndex];
                }
            }

            let barHeight = (max > 0 ? (matchCounts[index] / max) * maxBarHeight : 0);

            let barX = chartX + (index * (barPaddedWidth)) + (barPaddingX / 2);
            let barY = chartY - barHeight;

            let bar = chart.append("rect")
                .attr("x", barX)
                .attr("y", barY)
                .attr("width", barWidth)
                .attr("height", barHeight);

            if (index == maxIndex) {
                bar.attr("fill", "#a020a0");
            } else {
                bar.attr("fill", "#20a0a0");
            }
        }

        let xScale;
        let xTickFormat;

        switch (granularity) {
            case "years":
                xScale = d3.scaleBand().domain([selectedStartDate.getFullYear(), selectedStartDate.getFullYear() + d3.timeYear.count(selectedStartDate, selectedEndDate)]);
                break;

            case "months":
                xScale = d3.scaleBand().domain(d3.timeMonth.range(new Date(selectedStartDate.getFullYear(), selectedStartDate.getMonth()), d3.timeMonth.offset(new Date(selectedEndDate.getFullYear(), selectedEndDate.getMonth()), 1), 1));
                xTickFormat = d3.timeFormat("%B");
                break;

            case "days":
                xScale = d3.scaleTime().domain([selectedStartDate, selectedEndDate]);
                xTickFormat = d3.timeFormat("%d %B");
                break;
        }

        xScale.range([0, chartWidth]);

        let xAxis = d3.axisBottom(xScale);

        if (xTickFormat) {
            xAxis.tickFormat(xTickFormat);
        }

        chart.append("g")
            .attr("transform", `translate(${yAxisWidth}, ${xAxisHeight + chartHeight})`)
            .call(xAxis);

        let yScale = d3.scaleLinear().domain([0, max]).range([chartHeight, 0]);

        let yAxis = d3.axisLeft(yScale).tickFormat(d3.format(".0f"));

        chart.append("g")
            .attr("transform", `translate(${yAxisWidth}, ${xAxisHeight})`)
            .call(yAxis);
    }

    function drawGameModesChart(chart, gameModeCounts) {
        chart.selectAll("*").remove();

        let xAxisHeight = 40;
        let yAxisWidth = 40;

        let chartWidth = chart.node().clientWidth - 2 * yAxisWidth;
        let chartHeight = chart.node().clientHeight - 2 * xAxisHeight;
        let chartX = 0 + yAxisWidth;
        let chartY = 0 + chartHeight + xAxisHeight;

        chart.append("text")
            .attr("x", chartX - 25)
            .attr("y", chartY - chartHeight - 20)
            .attr("fill", "white")
            .attr("font-size", "0.8em")
            .text("MATCH COUNT")

        chart.append("text")
            .attr("x", chartX + (chartWidth / 2) - 40)
            .attr("y", chartY + 40)
            .attr("fill", "white")
            .attr("font-size", "0.8em")
            .text("GAME MODE")

        let max = 0;

        let radius = 6;

        let barWidth = 4;

        if (gameModeCounts.length > 0) {
            let maxIndex = 0;
            max = gameModeCounts[maxIndex];

            for (let i = maxIndex; i < gameModeCounts.length; i += 1) {
                if (gameModeCounts[i] > max) {
                    maxIndex = i;
                    max = gameModeCounts[maxIndex];
                }
            }

            let areaWidth = (chartWidth / gameModeCounts.length);

            let maxAreaHeight = chartHeight;

            for (let index = 0; index < gameModeCounts.length; index += 1) {
                let areaHeight = (max > 0 ? (gameModeCounts[index] / max) * maxAreaHeight : 0);

                let centerX = chartX + (index * (areaWidth)) + (areaWidth / 2);

                let barX = centerX - (barWidth / 2);
                let barY = chartY - areaHeight + radius;

                let barHeight = areaHeight;

                let centerY = barY - radius;

                let circle = chart.append("circle")
                    .attr("cx", centerX)
                    .attr("cy", centerY)
                    .attr("r", radius)
                    .attr("stroke-width", 2);

                let bar = chart.append("rect")
                    .attr("x", barX)
                    .attr("y", barY)
                    .attr("width", barWidth)
                    .attr("height", barHeight);

                if (index == maxIndex) {
                    circle.attr("stroke", "#d080d0").attr("fill", "#a020a0");

                    bar.attr("fill", "#a020a0");
                } else {
                    circle.attr("stroke", "#80d0d0")
                        .attr("fill", "#20a0a0");

                    bar.attr("fill", "#20a0a0");
                }
            }
        }

        let xScale = d3.scaleBand().domain(["CLASSIC", "ARAM", "SWIFTPLAY"]).range([0, chartWidth]);

        let xAxis = d3.axisBottom(xScale);

        chart.append("g")
            .attr("transform", `translate(${yAxisWidth}, ${xAxisHeight + chartHeight})`)
            .call(xAxis);

        let yScale = d3.scaleLinear().domain([0, max]).range([chartHeight, 0]);

        let yAxis = d3.axisLeft(yScale).tickFormat(d3.format(".0f"));

        chart.append("g")
            .attr("transform", `translate(${yAxisWidth}, ${xAxisHeight})`)
            .call(yAxis);
    }

    function drawMatchDurationsChart(chart, matches) {
        chart.selectAll("*").remove();

        let xAxisHeight = 40;
        let yAxisWidth = 40;

        let chartWidth = chart.node().clientWidth - 2 * yAxisWidth;
        let chartHeight = chart.node().clientHeight - 2 * xAxisHeight;
        let chartX = 0 + yAxisWidth;
        let chartY = 0 + chartHeight + xAxisHeight;

        chart.append("text")
            .attr("x", chartX - 25)
            .attr("y", chartY - chartHeight - 20)
            .attr("fill", "white")
            .attr("font-size", "0.8em")
            .text("DURATION [min]")

        chart.append("text")
            .attr("x", chartX + (chartWidth / 2) - 80)
            .attr("y", chartY + 40)
            .attr("fill", "white")
            .attr("font-size", "0.8em")
            .text("MATCH START TIME")

        let max = 0;

        let days = [];

        if (matches.length > 0) {
            let maxIndex = 0;
            max = matches[maxIndex].duration;

            for (let i = maxIndex; i < matches.length; i += 1) {
                if (matches[i].duration > max) {
                    maxIndex = i;
                    max = matches[maxIndex].duration;
                }
            }

            let barPaddedWidth = (chartWidth / matches.length);

            let barWidth = barPaddedWidth;

            let maxBarHeight = chartHeight;

            for (let index = 0; index < matches.length; index += 1) {
                let match = matches[index];

                days.push(match.start);

                let barHeight = (max > 0 ? (match.duration / max) * maxBarHeight : 0);

                let barX = chartX + (index * (barPaddedWidth));
                let barY = chartY - barHeight;

                let bar = chart.append("rect")
                    .attr("id", `mdBar${match.id}`)
                    .attr("x", barX)
                    .attr("y", barY)
                    .attr("width", barWidth)
                    .attr("height", barHeight)
                    .on("click", () => { matchDurationClick(match.id) });

                if (index == maxIndex) {
                    bar.attr("fill", "#a020a0");
                } else {
                    bar.attr("fill", "#20a0a0");
                }
            }
        }

        let xScale = d3.scaleBand().domain(days).range([0, chartWidth]);

        let skipFactor = Math.floor(days.length / 8);

        let xAxis = d3.axisBottom(xScale).tickValues(xScale.domain().filter((d, i) => !(i % skipFactor))).tickFormat(d3.timeFormat("%H:%M %d %b %Y"));

        chart.append("g")
            .attr("transform", `translate(${yAxisWidth}, ${xAxisHeight + chartHeight})`)
            .call(xAxis);

        let yScale = d3.scaleLinear().domain([0, Math.floor(max / 60)]).range([chartHeight, 0]);

        let yAxis = d3.axisLeft(yScale);

        chart.append("g")
            .attr("transform", `translate(${yAxisWidth}, ${xAxisHeight})`)
            .call(yAxis);
    }

    function drawTeamStats(chart, team) {
        let xAxisHeight = 40;
        let yAxisWidth = 40;

        let chartWidth = chart.node().clientWidth - 2 * yAxisWidth;
        let chartHeight = chart.node().clientHeight - 2 * xAxisHeight;

        let chartX = 0 + yAxisWidth;
        let chartY = 0 + chartHeight + xAxisHeight;

        chart.append("text")
            .attr("x", chartX - 25)
            .attr("y", chartY - chartHeight - 20)
            .attr("fill", "white")
            .attr("font-size", "0.8em")
            .text("AMOUNT")

        chart.append("text")
            .attr("x", chartX + (chartWidth / 2) - 40)
            .attr("y", chartY + 40)
            .attr("fill", "white")
            .attr("font-size", "0.8em")
            .text("USERNAME")

        let maxBarHeight = chartHeight;

        let names = [];

        let max = 0;

        let maxKIndex = 0;
        let maxK = team[maxKIndex].kills;

        let maxDIndex = 0;
        let maxD = team[maxDIndex].deaths;

        let maxAIndex = 0;
        let maxA = team[maxAIndex].assists;

        let i = 1;

        for (let index = 0; index < team.length; index += 1) {
            let summoner = team[index];

            if (!summoner.name) {
                summoner.name = `Unknown-${i}`;
                i += 1;
            }

            if (summoner.name in names) {
                summoner.name = `${summoner.name}-${i}`;
                i += 1;
            }

            names.push(summoner.name);

            if (summoner.kills > maxK) {
                maxKIndex = index;
                maxK = summoner.kills;
            }

            if (summoner.deaths > maxD) {
                maxDIndex = index;
                maxD = summoner.deaths;
            }

            if (summoner.assists > maxA) {
                maxAIndex = index;
                maxA = summoner.assists;
            }

            if (summoner.kills > max) {
                max = summoner.kills;
            }

            if (summoner.deaths > max) {
                max = summoner.deaths;
            }

            if (summoner.assists > max) {
                max = summoner.assists;
            }
        }

        let xScale = d3.scaleBand().domain(names).range([0, chartWidth]).paddingInner(0.2).paddingOuter(0.1);

        let areaWidth = xScale.bandwidth();

        for (let index = 0; index < team.length; index += 1) {
            let summoner = team[index];

            let barPaddedWidth = areaWidth / 3;
            let barPaddingX = 1;
            let barWidth = barPaddedWidth - barPaddingX;

            let barKHeight = (max > 0 ? (summoner.kills / max) * maxBarHeight : 0);
            let barDHeight = (max > 0 ? (summoner.deaths / max) * maxBarHeight : 0);
            let barAHeight = (max > 0 ? (summoner.assists / max) * maxBarHeight : 0);

            let areaX = xScale(summoner.name);

            let barKX = chartX + areaX;
            let barKY = chartY - barKHeight;

            let barDX = chartX + areaX + barPaddedWidth;
            let barDY = chartY - barDHeight;

            let barAX = chartX + areaX + 2 * barPaddedWidth;
            let barAY = chartY - barAHeight;

            let barK = chart.append("rect")
                .attr("x", barKX)
                .attr("y", barKY)
                .attr("width", barWidth)
                .attr("height", barKHeight);

            if (index == maxKIndex) {
                barK.attr("fill", "#b02020");
            } else {
                barK.attr("fill", "#602020");
            }

            let barD = chart.append("rect")
                .attr("x", barDX)
                .attr("y", barDY)
                .attr("width", barWidth)
                .attr("height", barDHeight);

            if (index == maxDIndex) {
                barD.attr("fill", "#2020b0");
            } else {
                barD.attr("fill", "#202060");
            }

            let barA = chart.append("rect")
                .attr("x", barAX)
                .attr("y", barAY)
                .attr("width", barWidth)
                .attr("height", barAHeight);

            if (index == maxAIndex) {
                barA.attr("fill", "#b0b020");
            } else {
                barA.attr("fill", "#606020");
            }
        }

        let xAxis = d3.axisBottom(xScale);

        chart.append("g")
            .attr("transform", `translate(${yAxisWidth}, ${xAxisHeight + chartHeight})`)
            .call(xAxis);

        let yScale = d3.scaleLinear().domain([0, max]).range([chartHeight, 0]);

        let yAxis = d3.axisLeft(yScale).tickFormat(d3.format(".0f"));

        chart.append("g")
            .attr("transform", `translate(${yAxisWidth}, ${xAxisHeight})`)
            .call(yAxis);
    }

    function drawSelectedMatch(t1finish, t1chart, t2finish, t2chart, match) {
        d3.select(t1finish).text("Select a team from the Match Durations panel").attr("style", "");
        t1chart.selectAll("*").remove();
        d3.select(t2finish).text("Select a team from the Match Durations panel").attr("style", "");
        t2chart.selectAll("*").remove();

        if (match) {
            if (match.win == 1) {
                d3.select(t1finish)
                    .text("VICTORY")
                    .attr("style", "font-size: 2em; color: #8080f0;");
                d3.select(t2finish)
                    .text("DEFEAT")
                    .attr("style", "font-size: 2em; color: #f08080;");
            } else {
                d3.select(t1finish)
                    .text("DEFEAT")
                    .attr("style", "font-size: 2em; color: #f08080;");
                d3.select(t2finish)
                    .text("VICTORY")
                    .attr("style", "font-size: 2em; color: #8080f0;");
            }

            let team1 = [];
            let team2 = [];

            for (let summoner of match.summoners) {
                if (summoner.team == 1) {
                    team1.push(summoner);
                } else {
                    team2.push(summoner);
                }
            }

            drawTeamStats(t1chart, team1);
            drawTeamStats(t2chart, team2);
        }
    }

    // END GRAPH DRAWING FUNCTIONS

    function matchDurationClick(matchId) {
        if (selectedMatch) {
            d3.select(`#mdBar${selectedMatch.id}`).attr("fill", previousColor);
        }

        for (let match of matches) {
            if (match.id == matchId) {
                selectedMatch = match;
                break;
            }
        }

        previousColor = d3.select(`#mdBar${matchId}`).attr("fill");

        d3.select(`#mdBar${matchId}`).attr("fill", "#206060");
    }

    // Matches over Time
    let matchCounts = $derived(countMatches(selectedMatches, granularity, selectedStartDate, selectedTimespanLength));

    let motDiv = $state(null);
    let motDivWidth = $state(0);
    let motDivHeight = $state(0);
    let motChart = $derived.by(
        () => {
            if (motDiv) {
                return d3.select(motDiv)
                    .select("svg")
                    .attr("width", motDivWidth)
                    .attr("height", motDivHeight);
            }
        }
    );

    $effect(
        () => {
            if (motChart) {
                drawMatchesOverTimeChart(motChart, matchCounts, granularity);
            }
        }
    );

    // Game Modes
    let gameModeCounts = $derived(countGameModes(selectedMatches));

    let gmDiv = $state(null);
    let gmDivWidth = $state(0);
    let gmDivHeight = $state(0);
    let gmChart = $derived.by(
        () => {
            if (gmDiv) {
                return d3.select(gmDiv)
                    .select("svg")
                    .attr("width", gmDivWidth)
                    .attr("height", gmDivHeight);
            }
        }
    );

    $effect(
        () => {
            if (gmChart) {
                drawGameModesChart(gmChart, gameModeCounts);
            }
        }
    );

    // Match Durations
    let mdDiv = $state(null);
    let mdDivWidth = $state(0);
    let mdDivHeight = $state(0);
    let mdChart = $derived.by(
        () => {
            if (mdDiv) {
                return d3.select(mdDiv)
                    .select("svg")
                    .attr("width", mdDivWidth)
                    .attr("height", mdDivHeight);
            }
        }
    );

    $effect(
        () => {
            if (mdChart) {
                drawMatchDurationsChart(mdChart, selectedMatches);
            }
        }
    );

    // Team 1
    let t1FinishDiv = $state(null);

    let t1StatsDiv = $state(null);
    let t1StatsDivWidth = $state(0);
    let t1StatsDivHeight = $state(0);
    let t1StatsChart = $derived.by(
        () => {
            if (t1StatsDiv) {
                return d3.select(t1StatsDiv)
                    .select("svg")
                    .attr("width", t1StatsDivWidth)
                    .attr("height", t1StatsDivHeight);
            }
        }
    );

    // Team 2
    let t2FinishDiv = $state(null);

    let t2StatsDiv = $state(null);
    let t2StatsDivWidth = $state(0);
    let t2StatsDivHeight = $state(0);
    let t2StatsChart = $derived.by(
        () => {
            if (t2StatsDiv) {
                return d3.select(t2StatsDiv)
                    .select("svg")
                    .attr("width", t2StatsDivWidth)
                    .attr("height", t2StatsDivHeight);
            }
        }
    );

    $effect(
        () => {
            drawSelectedMatch(t1FinishDiv, t1StatsChart, t2FinishDiv, t2StatsChart, selectedMatch);
        }
    )

    function resetAllFilters() {
        granularity = "days";

        resetTimespan();

        selectedMatch = null;
    }

    onMount(
		async () => {
			data = await d3.csv(
                "/src/data/league-data.csv",
                function(d) {
                    return {
                        game_id: d["game_id"],
                        game_start: parseDate(d["game_start_utc"]),
                        game_duration: + d["game_duration"],
                        game_mode: d["game_mode"],
                        champion_name: d["champion_name"],
                        summoner_id: d["summoner_id"],
                        summoner_name: d["summoner_name"],
                        team_id: d["team_id"],
                        win: !!(d["win"]),
                        team_position: d["team_position"],
                        kills: + d["kills"],
                        deaths: + d["deaths"],
                        assists: + d["assists"],
                    }
                }
            );

            await sleep(1000);

            loading = false;
		}
	);
</script>


<TopAppBar bind:this={topAppBar} variant="fixed">
    <Row>
        <Section>
            <TopAppBarTitle>League of Legends 2024 Match Data</TopAppBarTitle>
        </Section>
        <Section align="end" toolbar>
            <Button onclick={resetAllFilters}>
                <ButtonIcon class="material-icons">replay</ButtonIcon>
                <ButtonLabel>RESET ALL FILTERS</ButtonLabel>
            </Button>
        </Section>
    </Row>
</TopAppBar>

<AutoAdjust {topAppBar}>
    <Dialog bind:open={loading} scrimClickAction="" escapeKeyAction="">
        <DialogTitle>Loading data...</DialogTitle>
        <DialogContent>
            <LinearProgress indeterminate/>
        </DialogContent>
    </Dialog>

    <LayoutGrid>
        <Cell span={12}>
            <Paper>
                <PaperTitle>Matches over Time</PaperTitle>
                <PaperSubtitle>Time frame with the most matches is highlighted in purple</PaperSubtitle>
                <PaperContent>
                    <div style="display: flex; justify-content: center">
                        <p>Granularity of data:</p>
                    </div>
                    <div class="radio" style="display: flex; justify-content: center">
                        {#each granularityOptions as granularityOption}
                            <FormField>
                                <Radio bind:group={granularity} value={granularityOption.name}/>
                                {#snippet label()}
                                    {`${granularityOption.name[0].toUpperCase()}${granularityOption.name.slice(1)}`}
                                {/snippet}
                            </FormField>
                        {/each}
                    </div>

                    <br>

                    <div id="matches-over-time" class="graph" bind:this={motDiv} bind:clientWidth={motDivWidth} bind:clientHeight={motDivHeight}>
                        <svg></svg>
                    </div>

                    <br>

                    <div style="display: flex; justify-content: center;">
                        <p>Selected timespan from <b>{formatDate(d3.timeDay.offset(absoluteMinDate, sliderStartDateIndex))}</b> to <b>{formatDate(d3.timeDay.offset(absoluteMinDate, sliderEndDateIndex))}</b></p>
                    </div>
                    <Slider range bind:start={sliderStartDateIndex} bind:end={sliderEndDateIndex} min={minDateIndex} max={maxDateIndex} step={1} minRange={minRange}></Slider>
                    <div style="display: flex; justify-content: right; margin: 20px 10px 10px 10px;">
                        <Button variant="raised" onclick={resetTimespan} style="margin: 10px;">
                            <ButtonIcon class="material-icons">replay</ButtonIcon>
                            <ButtonLabel>RESET TIMESPAN</ButtonLabel>
                        </Button>
                    </div>
                </PaperContent>
            </Paper>
        </Cell>

        <Cell span={4}>
            <Paper>
                <PaperTitle>Game Modes</PaperTitle>
                <PaperSubtitle>
                    Most played game mode is highlighted in purple<br>
                    <br>
                </PaperSubtitle>
                <PaperContent>
                    <div id="game-modes" class="graph" bind:this={gmDiv} bind:clientWidth={gmDivWidth} bind:clientHeight={gmDivHeight}>
                        <svg></svg>
                    </div>
                </PaperContent>
            </Paper>
        </Cell>

        <Cell span={8}>
            <Paper>
                <PaperTitle>Match Durations</PaperTitle>
                {#if selectedMatch}
                    <PaperSubtitle>
                        Click on a duration bar to display match details below - selected match from {d3.timeFormat("%H:%M:%S %d %B %Y")(selectedMatch.start)}<br>
                        Longest match is highlighted in purple
                    </PaperSubtitle>
                {:else}
                    <PaperSubtitle>
                        Click on a duration bar to display match details below<br>
                        Longest match is highlighted in purple
                    </PaperSubtitle>
                {/if}
                <PaperContent>
                    <div id="match-durations" class="graph" bind:this={mdDiv} bind:clientWidth={mdDivWidth} bind:clientHeight={mdDivHeight}>
                        <svg></svg>
                    </div>
                </PaperContent>
            </Paper>
        </Cell>

        <Cell span={6}>
            <Paper>
                <PaperTitle>Team 1</PaperTitle>
                <PaperSubtitle>
                    Individual player statistics<br>
                    Kills are red, deaths are blue and assists are yellow
                </PaperSubtitle>
                <PaperContent>
                    <div style="display: flex; justify-content: center;">
                        <div id="team-1-finish" bind:this={t1FinishDiv}>"Select a team from the Match Durations panel"</div>
                    </div>
                    <div id="team-1-stats" style="height: 480px;" bind:this={t1StatsDiv} bind:clientWidth={t1StatsDivWidth} bind:clientHeight={t1StatsDivHeight}>
                        <svg></svg>
                    </div>
                </PaperContent>
            </Paper>
        </Cell>

        <Cell span={6}>
            <Paper>
                <PaperTitle>Team 2</PaperTitle>
                <PaperSubtitle>
                    Individual player statistics<br>
                    Kills are red, deaths are blue and assists are yellow
                </PaperSubtitle>
                <PaperContent>
                    <div style="display: flex; justify-content: center;">
                        <div id="team-2-finish" bind:this={t2FinishDiv}>"Select a team from the Match Durations panel"</div>
                    </div>
                    <div id="team-2-stats" style="height: 480px;" bind:this={t2StatsDiv} bind:clientWidth={t2StatsDivWidth} bind:clientHeight={t2StatsDivHeight}>
                        <svg></svg>
                    </div>
                </PaperContent>
            </Paper>
        </Cell>
    </LayoutGrid>
</AutoAdjust>


<style>
    :global(body) {
        padding: 0;
        margin: 0;
    }

    .radio > :global(*) {
        margin: 0.5em;
    }

    .graph {
        height: 280px;
    }
</style>
