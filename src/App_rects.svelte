<script>
  
  import { Bezier } from "bezier-js";
	import { onMount } from 'svelte';

  let canvas,
      ctx,
      BB, 
      zoom = 60,
      offsetX,
      offsetY,
      WIDTH, 
      HEIGHT,
      dragok = false,
      startX = 0,
      startY = 0,
      rects = [],
      meshSegments = [], 
      segmentPoints = [],
      pointSize = 6,
      pointFill = '#ff0000';

  export let mkAspect = "mk360";
  
  let mkAspects = [{name: "mk360", width: 1920, height: 1080}, {name: "mkpro", width: 1920, height: 1200}]
  let aspect = [mkAspects[0]]

  let meshResolutions = [{columns: 4, rows: 4}]
  let meshResolution = meshResolutions[0]

  rects.push({
      x: 75 - 15,
      y: 50 - 15,
      width: 30,
      height: 30,
      fill: "#444444",
      isDragging: false
  });
  rects.push({
      x: 75 - 25,
      y: 50 - 25,
      width: 30,
      height: 30,
      fill: "#ff550d",
      isDragging: false
  });
  rects.push({
      x: 75 - 35,
      y: 50 - 35,
      width: 30,
      height: 30,
      fill: "#800080",
      isDragging: false
  });
  rects.push({
      x: 75 - 45,
      y: 50 - 45,
      width: 30,
      height: 30,
      fill: "#0c64e8",
      isDragging: false
  });

  segmentPoints.push({
      x: 150,
      y: 40,
      width: pointSize*2,
      height: pointSize*2,
      fill: "#0c64e8",
      isDragging: false
  });

  segmentPoints.push({
      x: 80,
      y: 30,
      width: pointSize*2,
      height: pointSize*2,
      fill: "#0c64e8",
      isDragging: false
  });
  segmentPoints.push({
      x: 105,
      y: 150,
      width: pointSize*2,
      height: pointSize*2,
      fill: "#0c64e8",
      isDragging: false
  });

  

  const getSegmentPoints = (segmentPoints) => {
    let points = segmentPoints.map((point)=>{
      return [point.x,point.y] 
    })
    return points.flat()
  }

  const getAspect = (mkAspect) =>{
    let aspect = mkAspects.filter((aspect)=>{
      if(aspect.name == mkAspect){
        aspect.heightPercent = ( aspect.height * 100 ) / aspect.width
        return (aspect)
      }
    })
    return aspect
  }
	
  const drawGrid = (ctx) =>{
    ctx.strokeStyle = `#939393`;
    for (var x = 0, i = 0; i < meshResolution.columns; x+=(aspect[0].width/meshResolution.columns), i++) {
      ctx.beginPath();
      ctx.moveTo(x, 0);
      ctx.lineTo(x, aspect[0].height);
      ctx.stroke();   
      let tempSegment = {segmentPoints: []}
      for (var y = 0, j=0; j < meshResolution.rows; y+=(aspect[0].height/meshResolution.rows), j++) {            
        tempSegment.segmentPoints.push({
          x: x,
          y: y,
          width: pointSize*2,
          height: pointSize*2,
          fill: pointFill,
          isDragging: false
        })
      }
      meshSegments.push(tempSegment)
    }
    console.log(meshSegments)
    for (var y = 0, j=0; j < meshResolution.rows; y+=(aspect[0].height/meshResolution.rows), j++) {            
      ctx.beginPath();
      ctx.moveTo(0, y);
      ctx.lineTo(aspect[0].width, y);
      ctx.stroke();
    }
    ctx.fill();
    ctx.closePath();
  }
  /*
  const drawCurve = (ctx,curve) => {
    let steps = 100;
    let LUT = curve.getLUT(steps);
    for (let i = 1; i < steps; i++) {
      console.log(LUT[i - 1])
      ctx.strokeStyle = `#ff0000`;
      ctx.lineWidth = 3;
      ctx.beginPath();
      ctx.moveTo(LUT[i - 1].x, LUT[i - 1].y);
      ctx.lineTo(LUT[i].x, LUT[i].y);
      ctx.stroke();
    }
  };
  */

  function drawCurve(ctx, curve, offset) {
    offset = offset || { x:0, y:0 };
    var ox = offset.x;
    var oy = offset.y;
    ctx.beginPath();
    var p = curve.points, i;
    ctx.moveTo(p[0].x + ox, p[0].y + oy);
    if (p.length === 3) {
      ctx.quadraticCurveTo(
        p[1].x + ox, p[1].y + oy,
        p[2].x + ox, p[2].y + oy
      );
    }
    if (p.length === 4) {
      ctx.bezierCurveTo(
        p[1].x + ox, p[1].y + oy,
        p[2].x + ox, p[2].y + oy,
        p[3].x + ox, p[3].y + oy
      );
    }
    ctx.stroke();
    ctx.closePath();
  }

  function drawSkeleton(ctx, curve, offset = {x: 0, y: 0}, nocoords) {
    var pts = curve.points;
    ctx.strokeStyle = "lightgrey";
    drawLine(ctx, pts[0], pts[1], offset);
    if (pts.length === 3) drawLine(ctx, pts[1], pts[2], offset);
    else drawLine(ctx, pts[2], pts[3], offset);
    ctx.strokeStyle = "black";
    if(!nocoords) drawPoints(ctx, pts, offset);
  }

  function drawLine(ctx, p1, p2, offset) {
    offset = offset || { x:0, y:0 };
    var ox = offset.x;
    var oy = offset.y;
    ctx.beginPath();
    ctx.moveTo(p1.x + ox,p1.y + oy);
    ctx.lineTo(p2.x + ox,p2.y + oy);
    ctx.stroke();
  }

  function drawPoints(ctx, points, offset) {
    offset = offset || { x:0, y:0 };
    points.forEach(p => drawCircle(ctx, p, pointSize, offset));
  }

  function drawCircle(ctx, p, r, offset) {
    offset = offset || { x:0, y:0 };
    var ox = offset.x;
    var oy = offset.y;
    // ctx.strokeStyle = "#ff0000";
    ctx.beginPath();
    ctx.arc(p.x + ox, p.y + oy, r, 0, 2*Math.PI);
    ctx.fillStyle = p.fill;
    ctx.fill();
    ctx.strokeStyle = '#003300';
  }

  function rect(x, y, w, h) {
      ctx.beginPath();
      ctx.rect(x, y, w, h);
      ctx.closePath();
      ctx.fill();
  }

  function clear() {
    ctx.clearRect(0, 0, WIDTH, HEIGHT);
  }

  function draw() {
    clear();
    ctx.fillStyle = "#FAF7F8";
    rect(0, 0, WIDTH, HEIGHT);
    drawGrid(ctx)
    let curve = new Bezier(getSegmentPoints(segmentPoints));
    drawCurve(ctx,curve)
    drawSkeleton(ctx,curve)
    // redraw each rect in the rects[] array
    for (var i = 0; i < rects.length; i++) {
        var r = rects[i];
        ctx.fillStyle = r.fill;
        rect(r.x, r.y, r.width, r.height);
    }
    for (var i = 0; i < segmentPoints.length; i++) {
        var r = segmentPoints[i];
        // ctx.fillStyle = r.fill;
        // rect(r.x, r.y, r.width, r.height);
        // drawCircle(ctx, {x: (r.x-(r.width/2)), y: (r.y-(r.height/2)), fill:"#ffcc00" }, pointSize)
        drawCircle(ctx, {x: (r.x), y: (r.y), fill:"#ffcc00" }, pointSize)
    }
  }

  function calcPercent(ex,ey) {
    let xPc = (ex)/BB.width;
    let yPc = (ey)/BB.height;
    let mx = aspect[0].width*xPc;
    let my = aspect[0].height*yPc;
     return { mx, my }
  }

  function myDown(e) {
    // tell the browser we're handling this mouse event
    e.preventDefault();
    e.stopPropagation();
    // get the current mouse position
    let tempx = parseInt(e.clientX - offsetX);
    var tempy = parseInt(e.clientY - offsetY);
    let {mx,my} = calcPercent(tempx,tempy);
    // test each rect to see if mouse is inside
    dragok = false;
    for (var i = 0; i < rects.length; i++) {
        var r = rects[i];
        if (mx > r.x && mx < r.x + r.width && my > r.y && my < r.y + r.height) {
            // if yes, set that rects isDragging=true
            dragok = true;
            r.isDragging = true;
        }
    }
    for (var i = 0; i < segmentPoints.length; i++) {
        var r = segmentPoints[i];
        if (mx > (r.x-pointSize) && mx < (r.x+pointSize) && my > (r.y-pointSize) && my < (r.y+pointSize)) {
            // if yes, set that rects isDragging=true
            console.log(r)
            dragok = true;
            r.isDragging = true;
        }
    }
    // save the current mouse position
    startX = mx;
    startY = my;
  }

  function myUp(e) {
    // tell the browser we're handling this mouse event
    e.preventDefault();
    e.stopPropagation();
    // clear all the dragging flags
    dragok = false;
    for (var i = 0; i < rects.length; i++) {
      rects[i].isDragging = false;
    }
    for (var i = 0; i < segmentPoints.length; i++) {
      segmentPoints[i].isDragging = false;
    }
  }

  function myMove(e) {
    // if we're dragging anything...
    if (dragok) {
        // tell the browser we're handling this mouse event
        e.preventDefault();
        e.stopPropagation();
        // get the current mouse position
        let tempx = parseInt(e.clientX - offsetX);
        var tempy = parseInt(e.clientY - offsetY);
        let {mx,my} = calcPercent(tempx,tempy);
        // calculate the distance the mouse has moved
        // since the last mousemove
        var dx = mx - startX;
        var dy = my - startY;
        // move each rect that isDragging
        // by the distance the mouse has moved
        // since the last mousemove
        for (var i = 0; i < rects.length; i++) {
            var r = rects[i];
            if (r.isDragging) {
                r.x += dx;
                r.y += dy;
            }
        }
        for (var i = 0; i < segmentPoints.length; i++) {
            var r = segmentPoints[i];
            if (r.isDragging) {
                r.x += dx;
                r.y += dy;
            }
        }
        // redraw the scene with the new rect positions
        draw();
        // reset the starting mouse position for the next mousemove
        startX = mx;
        startY = my;
    }
  }

	onMount(() => {
    aspect = getAspect(mkAspect)
		ctx = canvas.getContext('2d');
    setTimeout(()=>{
      BB = canvas.getBoundingClientRect();
      offsetX = BB.left;
      offsetY = BB.top;
      WIDTH = canvas.width;
      HEIGHT = canvas.height;
      canvas.onmousedown = myDown;
      canvas.onmouseup = myUp;
      canvas.onmousemove = myMove;
      draw()
    },1000)
    
	});
</script>

<div class="mesh_container">
  <canvas
    bind:this={canvas}
    width={aspect[0].width}
    height={aspect[0].height}
    style="--zoom:{zoom}%"
  ></canvas>
</div>

<style lang="scss">
  :global(html,body) {
		margin: 0;
    padding: 0;
    width: 100%;
    height: 100%;
	}
  :global {
    * { 
      box-sizing: border-box;
    }
    #app {
      margin: 0;
      padding: 0;
      width: 100%;
      height: 100%;
    }
	}
  .mesh_container {
    position: relative;
    width: 100%;
    height: 100%;
    display: flex;
    align-items: center;
    justify-content: center;
  }
	canvas {
    width: var(--zoom);
		background-color: #dfdfdf;
	}
</style>