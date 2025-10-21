<script lang="ts">

	// Imports copied from the template.
	import * as d3 from 'd3';

	// Item[] types copied from the template.
	interface Item {
		city: string;
		country: string;
		mainPollutant: string;
		pm25: number;
		state: string;
		stationName: string;
		timestamp: Date;
		usAqi: number;
	}

	// Properties this component accepts: added showRawPoints
	const { data, showRawPoints, colorMap }: {
		data: Item[];
		showRawPoints: boolean;
		colorMap: Record<string, string>;
	} = $props();

	// AQI constants copied from assignment page.
	const AQI_LEVELS = [
		{ name: "Good", min: 0, max: 50, color: "#9cd84e" },
		{ name: "Moderate", min: 51, max: 100, color: "#facf39" },
		{ name: "Unhealthy for Sensitive Groups", min: 101, max: 150, color: "#f99049" },
		{ name: "Unhealthy", min: 151, max: 200, color: "#f65e5f" },
		{ name: "Very Unhealthy", min: 201, max: 300, color: "#a070b6" },
		{ name: "Hazardous", min: 301, max: 9999, color: "#a06a7b" }
	];

	// Let svg be this SVG type. Instead of adding touching <svg> elements,
	// I perfer appending things to keep things more comprehendable for myself due to references from d3 examplars.
	let svg: SVGSVGElement;
	
	// I shoved this piece from Lab 7.
	let width = $state(1200), height = $state(600);

	// Margin but shows 1 pixel outside of the domains.
	let margin = { top: 1, right: 0, bottom: 250, left: 30 };

	// An array of object for getting all the monthly data into one after processing the data.
	let aggregated: {
		date: Date;
		mean: number;
		p10: number;
		p90: number;
	}[] = [];
	
	// Radius for controlling when hover is on or not.
	const hoverRadius = 20;

	// Tooltip size experimented by nudging.
	// Ignore width number, it is covered anyways. (too lazy to change)
	const tipWidth = 270, tipHeight = 50, tipPad = 8, offsetX = 12, offsetY = 12;

	// Aggregation by day and mapped initialization.
	let aggregatedByDay: {
		 date: Date; 
		 values: Item[];
	}[] = [];

	let byDayMap: Map<number, Item[]> = new Map();

	function processData() {

		// Grouping every row's date that returns the same year + month + days (that normalized to 15).
		// In a format of Mapped array with normalized Dates as keys.
		const grouped = d3.group(data, d => {
			const date = new Date(d.timestamp);
			return new Date(date.getFullYear(), date.getMonth(), 15);
		});

		// Aggregate the data for line & area
		aggregated = Array.from(grouped, ([key, values]) => {

			// Grab by remaping values made of all of these in Item[], into just keys with array of usAqi numbers.
			// Also sorted for quantile function :P
			const aqiValues = values
				.map(d => d.usAqi)
				.sort(d3.ascending);

			// Return object that has these.
			// Also I make sure that numbers are not undefined (if so becomes 0), to keep the data type of aggregated happy.
			return {
				date: new Date(key),
				mean: d3.mean(aqiValues) ?? 0,
				p10: d3.quantile(aqiValues, 0.1) ?? 0,
				p90: d3.quantile(aqiValues, 0.9) ?? 0
			};
		});

		// Welp, some visual bugs from using All Station data causes the lines starting from the middle of the chart.
		// I suspected that it was due to the aggreated data not being sorted by time, and it works.
		aggregated.sort((a, b) => a.date.getTime() - b.date.getTime());

		// Grouped by Day instead of 15th and Remapped like above.
		const byDay = d3.group(data, d => d3.utcDay.floor(d.timestamp));
		aggregatedByDay = [];
  		byDayMap = new Map();

		for (const [date, values] of byDay) {
			aggregatedByDay.push({ date, values });
		}

		aggregatedByDay.sort((a, b) => a.date.getTime() - b.date.getTime());
  		byDayMap = new Map(aggregatedByDay.map(g => [g.date.getTime(), g.values]));

	}

	let zoomTransform: d3.ZoomTransform = d3.zoomIdentity;

	function drawChart() {

		// Ah good old select and remove all if it has to redraw. Man, I miss 15104.
		d3.select(svg).selectAll("*").remove();

		// New Width and Height changed by margins, for graph not axis.
		let innerWidth = width - margin.left - margin.right;
		let innerHeight = height - margin.top - margin.bottom;

		// Initialize svg that with width height and translate origin beased on margin and top
		let g = d3.select(svg)
			.attr("width", width)
			.attr("height", height)
			.append("g")
			.attr("transform", `translate(${margin.left},${margin.top})`);

		// Gets the maximum and minimum date based on dates.
		const Extent = d3.extent(data, d => d.timestamp);
        const ExtentAgg = d3.extent(aggregated, d => d.date);

		// Set bot and top for the date, in which will contain everything.
        let bot : Date;
        let top : Date;

		// Get minimum for bot.
        if (Extent[0] && ExtentAgg[0])
        {
            bot = new Date(Math.min(Extent[0].getTime(), ExtentAgg[0].getTime()));
        }
        else
        {
            bot = new Date(2002, 11, 24); //Yes, it's my birthday.
        }

		// Get maximum for top.
        if (Extent[1] && ExtentAgg[1])
        {
            top = new Date(Math.max(Extent[1].getTime(), ExtentAgg[1].getTime()));
        }
        else
        {
            top = new Date(2002, 11, 24); //Yes, it's my birthday.
        }

		// Use top and bot for X-axis domain.
        const xDomain: [Date, Date] = [bot, top];

		// Scale time UTC (because of data) from xDomain to [0, innerWidth].
		let x = d3.scaleUtc().domain(xDomain).range([0, innerWidth]);

		// Get maximum AQI, return 0 if undefined to keep things happy.
		let maxAqi = d3.max(data, d => d.usAqi) ?? 0;

		// Scale range of AQI (can't be negative + some buffer to show circles) to [innerHeight, 0].
		const y = d3.scaleLinear().domain([0, maxAqi + 5]).range([innerHeight, 0]);

		// Background AQI Color for each level
		AQI_LEVELS.forEach(level => {

			// Set level height so that if the first level is slected (min == 0) make height 1 less so it doesn't go beyond the y Axis.
			let levelHeight = y(level.min) - y(level.max + 1);
			if (level.min == 0)
			{
				levelHeight = y(level.min) - y(level.max);
			}

			// Create rectangle.
				g.append("rect")
					.attr("x", 0)
					.attr("y", y(level.max))
					.attr("width", innerWidth) 		       // Entire width of inner graphing area.
					.attr("height", levelHeight)
					.attr("fill", level.color)
					.attr("opacity", 0.5) 				   // I guess the example has this opacity?
		});

		// Area: 10 to 90 percentile
		// reference: https://observablehq.com/@d3/area-chart/2. 
		const area = d3.area<{ date: Date; p10: number; p90: number }>()       // Have to tell TypeScrpt their types again.
			.x(d => x(d.date))
			.y0(d => y(d.p10))
			.y1(d => y(d.p90));

		const areaDraw = g.append("path")
			.attr("class", "area")
			.attr("fill", "grey")
			.attr("opacity", 0.3)
			.attr("d", area(aggregated));

		// Line: average AQI
		const line = d3.line<{ date: Date; mean: number }>()
			.x(d => x(d.date))
			.y(d => y(d.mean));

		const meanLine = g.append("path")
			.attr("class", "mean-line") 
			.attr("fill", "none")
			.attr("stroke", "black")
			.attr("stroke-width", 2)
			.attr("d", line(aggregated));

		// Remove previous raw points (if any).
		g.selectAll(".raw-point").remove();

		const y_grid = g.append("g")
			.attr("class", "y-grid")
			.call(
				d3.axisLeft(y)						       // Size of width but the right direction.
				.tickSize(-innerWidth)
			);

		y_grid.selectAll("text").remove();             	   // Remove text labels.
		y_grid.selectAll("line")					       // Replicate grid line from example.
			.attr("stroke", "gray")
			.attr("opacity", 0.3);
		y_grid.select(".domain").remove();                 // Remove black lines created by domain class in <path>.
		
		// ----- Part 2 Functions -----

		// * Function 0: Tooltip *

		// Create a hover laver for detecting where you mouse hover.
		const hoverLayer = g.append("g")
			.attr("class", "hover-layer")
			.style("pointer-events", "none");

		// Setting for the dashline that you see aligned with data points.
		const hoverLine = hoverLayer.append("line")
			.attr("class", "hover-line")
			.attr("y1", 0)
			.attr("y2", innerHeight)
			.attr("stroke", "steelBlue")
			.attr("stroke-dasharray", "4,4")
			.attr("stroke-width", 2)
			.attr("opacity", 0);

		// Tip Setup
		const tip = hoverLayer.append("g")
			.attr("opacity", 0);

		// We’ll recreate text lines per hover (so clear the old ones each time)
		let currentHoverIndex: number | null = null;

		// Function that toggles whether it is going to show or not.
		const showHover = (show: boolean) => {
			let o = 0;
			if (show) {o = 1;}
			hoverLine.attr("opacity", o);
			tip.attr("opacity", o);
		};

		// I moved circle axis and hide here to make it above the line

		const dataCircle = g.append("g")
			.append("g")
			.selectAll("circle")					   	   // Reference: https://d3js.org/d3-selection/joining#selection_enter.
			.data(data)                                	   // Basically equivalant to <#each> in <g>
			.enter()
			.append("circle")
			.attr("class", "raw-point")					   // For removal of all raw-points, thus classed.
			.attr("cx", d => x(d.timestamp))
			.attr("cy", d => y(d.usAqi))
			.attr("r", 1.5)
			.attr("fill", "black")
			.attr("opacity", 1);

		// Conditionally render cirlces (invisible if radius is 0).
		if (!showRawPoints) {
			dataCircle.attr("r", 0);
		}

		// Cover whatever is the graphic is behind. (Yes, im too lazy to limit their boundaries)
		const whiteRect = g.append("rect")
			.attr("class", "add-on")
			.attr("x", -margin.left)
			.attr("y", 0)
			.attr("width", margin.left)
			.attr("height", innerHeight)
			.attr("fill", "white");

		// Default y axis, I think it is what the example giving us, exacpt the domain for y-axis is removed.
		const y_axis = g.append("g")
			.attr("class", "y-axis")
			.call(d3.axisLeft(y));

		y_axis.select(".domain").remove();

		const x_axis = g.append("g")
			.attr("class", "x-axis") 
			.attr("transform", `translate(0,${innerHeight})`)
			.call(d3.axisBottom(x)
				.tickFormat(d => d3.utcFormat("%b %d %Y")(d as Date))			   // Format it to MON XX 20XX UTC to line up.
				.ticks(6)								   					   // Make ticks less dense, roughly at 6.
			);

		// Welp friends suggested that I could have just make a function that take any as a type for function.
		// In which I could have used transformation of zx from zoom here.
		// Too lazy to change whatever if before, so here it is.
		const positionHoverForDate = (group: { date: Date; values: Item[] }, zx: any) => {

			// Tranform based on zoom.
			const cx = zx(group.date);

			// Set the end points' x position to be whatever is transformed.
			hoverLine
				.attr("x1", cx)
				.attr("x2", cx);

			// Clear all tip's children before redrawing.
			tip.selectAll("*").remove();

			// Sort group's balue based on each date's different station's usAQI value, for visual purposes.
			const sortedValues = group.values.sort((a, b) => d3.descending(a.usAqi, b.usAqi));

			// Show selected date's <rect> as header
			const headerH = 24;							   // Height nugdged to 24, adapting to default font size.

			tip.append("rect")
				.attr("x", 0)
				.attr("y", 0)
				.attr("width", tipWidth)
				.attr("height", headerH)
				.attr("rx", 8)							   // Rounded edges.
				.attr("ry", 8)
				.attr("fill", "white")
				.attr("stroke", "black")
				.attr("opacity", 0.75);

			tip.append("text")
				.attr("x", tipPad)
				.attr("y", headerH / 2)					   // For centering.
				.attr("font-weight", "bold")
				.attr("dominant-baseline", "middle")       // Reference: https://developer.mozilla.org/en-US/docs/Web/SVG/Reference/Attribute.
				.text(d3.utcFormat("%b %d %Y")(group.date));

			// These are some sizes for each item rectangles formatting.
			const itemFS = 10;							   // Font size.
			const itemPad = 2;							   // Padding in <rect>.
			const gap = 4;								   // Gab between <rect>.
			const itemH = itemPad * 2 + itemFS;			   // <rect> heights.

			// Noticed Pittsburgh data not having station names thus => make it a format of city, state.
			const makeLabel = (d: Item) => {
				if (!d.stationName)
				{
					return `${d.country || "Unknown"}-${d.state || "Unknown"}-${d.city || "Unknown"}`
				} 
				return d.stationName;
			}

			// Array of strings for station labels.
			const labels = sortedValues.map(makeLabel);


			// Build a stable domain of station names once

			// Get item, then get station name, then transform, then use as key to get color.
			const colorFor = (d: Item) => {
				let key = d.stationName;
				if (!d.stationName)
					{
						key = `${d.country || "Unknown"}-${d.state || "Unknown"}-${d.city || "Unknown"}`
					} 
				return colorMap[key];
			};

			// Measuring the size of the text in pixels. (Outsie of the screen)
			const measurer = tip.append("text")
				.attr("x", -69420)						   // Outside.
				.attr("y", -69420)
				.attr("font-size", itemFS)
				.attr("opacity", 0);

			function measure(text: string) {
				measurer.text(text);
				const node = measurer.node();
				if (node != null)						   // Keeping TypeScript Happy.
				{
					return node.getComputedTextLength() as number;
				}
				return 0;
			}

			// Maximum with for each station Column.
			const maxLabelW = d3.max(labels, s => measure(s)) ?? 0;

			// I'm gonna use tspan to control the columns of text to align better. These are some pixels for columns to start.
			const colStation = tipPad;
			const colAqi = colStation + maxLabelW + 16;
			const aqiSampleW = measure("AQI: 000");		   // AQI can't go above 999 right?
			const colPm = colAqi + aqiSampleW + 8;
			const pmSampleW = measure("PM2.5: 000.00");	   // PM2.5 can't go above 999.99 (2 significant value).

			// Better sizing of final with for each tip.
			const endWidth = colPm + pmSampleW+ 12;

			measurer.remove(); 							   // Cleanup measurer.

			// For each Item type from sortedValues draw a rectangle that contains these.
			sortedValues.forEach((d, i) => {

				// Locators.
				const yTop = headerH + gap + i * (itemH + gap);				   // For <rect>.
				const yMid = yTop + itemH / 2;								   // For <text>.

				const sColor = colorFor(d);				   					   // Station Color

				tip.append("rect")
				.attr("x", 0)
				.attr("y", yTop)
				.attr("width", tipWidth)
				.attr("height", itemH)
				.attr("rx", 8)
				.attr("ry", 8)
				.attr("fill", "white")
				.attr("stroke", sColor)
				.attr("opacity", 0.9);
				
				// Reset Circle
				dataCircle
					.attr("r", 1.5)
					.attr("stroke", null)
					.attr("stroke-width", null)
					.attr("fill", "black");

				const dayKey = group.date.getTime();

				// Make circle large and matching color.
				dataCircle
					.filter(p => d3.utcDay(p.timestamp).getTime() === dayKey)
					.attr("r", 4)
					.attr("stroke", d => colorFor(d))
					.attr("stroke-width", 1.5)
					.attr("fill", d => colorFor(d));
							

				// Apply locator + style to text.
				const t = tip.append("text")
					.attr("font-size", itemFS)
					.attr("fill", sColor)
					.attr("y", yMid)
					.attr("dominant-baseline", "middle")

				// Station name bolded.
				t.append("tspan")
					.attr("x", colStation)
					.style("font-weight", "bold")
					.text(makeLabel(d));

				// AQI title bolded.
				t.append("tspan")
					.attr("x", colAqi)
					.style("font-weight", "bold")
					.text(`AQI:`);

				// AQI value not bolded.
				t.append("tspan")
					.attr("dx", 4)
					.text(`${d.usAqi}`);

				// PM2.5 title bolded.
				t.append("tspan")
				.attr("x", colPm)
				.style("font-weight", "bold")
				.text(`PM2.5:`);

				// PM2.5 value not bolded.
				t.append("tspan")
					.attr("dx", 4)
					.style("font-weight", null)
					.text(String(d.pm25));
			});

			// Translate the tip to either left or right of line with certain pixels.
			let tx = cx + offsetX
			let ty = tipPad;

			// If larger than innerWidth flip by moving tx all the way to the left.
			if (tx + tipWidth > innerWidth) 
			{
				tx = cx - offsetX - tipWidth;
			}

			tip.attr("transform", `translate(${tx},${ty})`);

			// Wrap with new width.
			tip.selectAll("rect").attr("width", endWidth);
		}

		// Welp I only uses when zoom calls it so I don't think I want to keep TypseScript happy but cheating with any.
		const mouseHover = (sel: any) => {
			sel
				.on("pointermove", (event: PointerEvent) => { 				   // Welp pointerevent unlike mouse event will capture drag.
					
					// Get transformed zx from zoom.
					const zx = zoomTransform.rescaleX(x);

					// Get x from mouse.
					const [mx] = d3.pointer(event, g.node() as SVGGElement);

					// Gets index of based on mouse's x. Reference: https://d3js.org/d3-array/bisect.
					const i = d3.bisector((d: { date: Date }) => d.date).center(aggregatedByDay, zx.invert(mx));
					
					// Closeset aggregated by day.
					const closestAgg = aggregatedByDay[i];

					const dx = Math.abs(zx(closestAgg.date) - mx); 			   // Pixel distance of mouse to that point in x tho.

					// Within tolerence than do make tips.
					if (dx <= hoverRadius) 
					{
						currentHoverIndex = i;
						positionHoverForDate(closestAgg, zx);
						showHover(true);
					} 
					
					// If not hide renderings and reset index.
					else 
					{
						// Reset Circle
						dataCircle
							.attr("r", 1.5)
							.attr("stroke", null)
							.attr("stroke-width", null)
							.attr("fill", "black");
						showHover(false);
						currentHoverIndex = null;
					}
				})

				// If mouse leave the area hide and reset.
				.on("mouseleave", (event: MouseEvent) => {
					//Reset Cirlce
					dataCircle
						.attr("r", 1.5)
						.attr("stroke", null)
						.attr("stroke-width", null)
						.attr("fill", "black");
					showHover(false);
					currentHoverIndex = null;
				});
		};

			// * Function 2: Pie Chart - Semi Brush *

			// Pie Translate amount by x and y.
			const pieCX = 45;
			const pieCY = innerHeight + 120;
			
			// Pie Radius.
			const pieR  = 60;

			// Add pie + translate.
			const pieG = g.append("g")
				.attr("class", "pie")
				.attr("transform", `translate(${pieCX},${pieCY})`);

			// Pie tile 12 pixels above the chart.
			const pieTitle = g.append("text")
				.attr("class", "pie-title")
				.attr("x", pieCX - pieR )
				.attr("y", pieCY - pieR - 12)
				.attr("text-anchor", "left")
				.attr("font-weight", "bold");

			// Side text 12 pixels right of the pie chart.
			const pieSide = g.append("g")
				.attr("class", "pie-side")
				.attr("transform", `translate(${pieCX + pieR + 12}, ${pieCY})`);

			// The text that shows the zoom brush doamin.
			const pieSideRange = pieSide.append("text")
				.attr("class", "pie-range")
				.attr("dominant-baseline", "middle")
				.attr("font-size", 12)
				.attr("fill", "Black");
			
			// Details of what each color of the pie represents. 32 pixels bellow range text
			const pieSideInfo = pieSide.append("text")
				.attr("class", "pie-info")
				.attr("font-size", 12)
				.attr("fill", "Black")
				.attr("dy", 32)
				.style("opacity", 0);

			// For every AQI level find the number within the boundary.
			const levelOf = (aqi: number) => AQI_LEVELS.find(L => aqi >= L.min && aqi <= L.max);

			const levelByName = (name: string) => AQI_LEVELS.find(L => L.name === name); 

			// Make the pi.
			const pie = d3.pie<{ name: string; color: string; count: number }>()
				.value(d => d.count);

			// One base arc + one hover to enlarge with a factor of 1.07.
			const arcBase = d3.arc<d3.PieArcDatum<{ name: string; color: string; count: number }>>()
				.innerRadius(0)
				.outerRadius(pieR);

			const arcHover = d3.arc<d3.PieArcDatum<{ name: string; color: string; count: number }>>()
				.innerRadius(0)
				.outerRadius(pieR * 1.07);

			// Update pie based on the zoomed x-scale (zx), and decault x has the same formate as well.
			const updatePie = (zx: d3.ScaleTime<number, number>) => {

				// Get domain of zx.
				const [x0, x1] = zx.domain();

				// Type the date range.
				pieSideRange.text(`Date Range: ${d3.utcFormat("%b %d %Y")(x0)} – ${d3.utcFormat("%b %d %Y")(x1)}`);

				pieSideInfo.style("opacity", 0).text("");

				// Const for all data 
				const visible = data.filter(d => d.timestamp >= x0 && d.timestamp <= x1);

				// Total data selected
				const total = visible.length;

				// Build buckets in AQI order, as an array of objects with name, color and counts.
				const buckets = AQI_LEVELS.map(L => ({ name: L.name, color: L.color, count: 0 }));

				for (const d of visible) {
					const L = levelOf(d.usAqi);
					if (L)
					{
						const b = buckets.find(b => b.name === L.name)!;
						b.count++;
					}
			}

			// Title of Pie.
			pieTitle.text(`Selected records: ${total}`);

			// Binding data to deal with enter, update, exit. 
			const arcs = pieG
				// Select all paths within pieG, with types of SVGPathElement and data type of pie arc datum of buckets array element.
				.selectAll<SVGPathElement, d3.PieArcDatum<typeof buckets[number]>>("path")

				// Data becomes the pie of buckets, with key as name.
				.data(pie(buckets), d => d.data.name)

				// The JOIN.
				.join(
					enter => enter.append("path")
						.attr("fill", d => d.data.color)
						.attr("d", arcBase),
					update => update
						.attr("fill", d => d.data.color)
						.attr("d", arcBase),
					exit => exit.remove()
				);

			// Hover effect that includes but enlarge and strink.
			arcs
				.on("mouseenter", (event, d) => {
					
					// Select the current target that mouseenter reads.
					d3.select(event.currentTarget)
						.transition().duration(180).ease(d3.easeCubicOut)
						.attr("d", arcHover(d));

					// Select label within the selected then grab allslices data name that mataches the selected.
					const lbl = pieG.selectAll<SVGTextElement, typeof d>("text.slice-label")
						.filter(ld => ld.data.name === d.data.name);

					// Move the label to hovered centroid.
					lbl.transition().duration(180).ease(d3.easeCubicOut)
						.attr("transform", `translate(${arcHover.centroid(d)})`);

					// Show category info under the date range
					const L = levelByName(d.data.name);						   // Get level by name.

					// Display the AQI range and count for the hovered slice.
					if (L) {
						pieSideInfo
							.text(`${d.data.name} (${L.min}–${L.max}): ${d.data.count}`)
							.style("opacity", 1);
					}
				})

				// If leave, do the same but reverse and hide.
				.on("mouseleave", (event, d) => {
					d3.select(event.currentTarget)
						.transition().duration(180).ease(d3.easeCubicOut)
						.attr("d", arcBase(d));

					const lbl = pieG.selectAll<SVGTextElement, typeof d>("text.slice-label")
						.filter(ld => ld.data.name === d.data.name);

					lbl.transition().duration(180).ease(d3.easeCubicOut)
						.attr("transform", `translate(${arcBase.centroid(d)})`);

					// Hide pieSideInfo
					pieSideInfo.style("opacity", 0);
				});

			// Labels for slices with nonzero counts
			const labels = pieG
				.selectAll<SVGTextElement, d3.PieArcDatum<typeof buckets[number]>>("text.slice-label")
				.data(pie(buckets).filter(d => d.data.count > 0), d => d.data.name);     // Filter zero counts.

				labels.join(
					enter => enter.append("text")
						.attr("class", "slice-label")
						.attr("pointer-events", "none")
						.attr("text-anchor", "middle")
						.attr("font-size", 8)
						.attr("fill", "Black")
						.attr("transform", d => `translate(${arcBase.centroid(d)})`)
						.text(d => `${d.data.count}`),
					update => update
						.attr("transform", d => `translate(${arcBase.centroid(d)})`)
						.text(d => `${d.data.count}`),
					exit => exit.remove()
				);

			};

			updatePie(x);



		// * Function 3: Zoom *

		// Multiplier for better control of zooming.
		let zoomScale = (top.getTime() - bot.getTime()) * .000000001;

		// Zoom behavior objects, thus attached to a <rect> element, no data bound.
		// Reference: https://observablehq.com/@d3/pan-zoom-axes?collection=@d3/d3-zoom.
		const zoom = d3.zoom<SVGRectElement, unknown>()
			.scaleExtent([1, zoomScale])
			.translateExtent([[-100, 0], [innerWidth + 100, 0]])			   // You can only pan to -100 and width + 100. No vertically cuz not necessary.
			.extent([[0, 0], [innerWidth, innerHeight]])
			.on("zoom", (event: d3.D3ZoomEvent<SVGRectElement, unknown>) => {  // As you can see I don't like TypeScript.
				

				zoomTransform = event.transform;

				const zx = zoomTransform.rescaleX(x);
				const zy = y;												   // Kept same.

				// Update x-axis
				x_axis.call(d3.axisBottom(zx)
						.tickFormat(d => d3.utcFormat("%b %d %Y")(d as Date))
						.ticks(6)
				);

				// Update Average
				const lineZoomed = d3.line<{ date: Date; mean: number }>()
					.x(d => zx(d.date))
					.y(d => zy(d.mean));

				meanLine
					.attr("d",lineZoomed(aggregated));

				// Update Area
				const areaZoomed = d3.area<{ date: Date; p10: number; p90: number }>()
					.x(d => zx(d.date))
					.y0(d => zy(d.p10))
					.y1(d => zy(d.p90));

				areaDraw
					.attr("d",areaZoomed(aggregated));

				// Update Points
				dataCircle
					.attr("cx", d => zx(d.timestamp));
				
				updatePie(zx);
			});

			// Create the transparent rectangle for actions like panning and zooming to happen.
			g.append("rect")
				.attr("class", "zoom-overlay")
				.attr("x", 0).attr("y", 0)
				.attr("width", innerWidth)
				.attr("height", innerHeight)
				.style("fill", "none")
				.style("pointer-events", "all")
				.call(zoom)
				.call(zoom.transform, zoomTransform)      // Apply transformation from last stored transformation.
				.call(mouseHover);






	}


	
	// Whenever data changes, process and draw.
	$effect(() => {

		if (svg && data && data.length > 0) {
			processData();
			drawChart();
		}
		
	});

</script>

<svg bind:this={svg}></svg>

<style>
	svg {
		font-family: sans-serif;
	}
</style>