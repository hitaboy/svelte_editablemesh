<script>
  
  import { Bezier } from "bezier-js";
	import { onMount } from 'svelte';

  let canvas,
      ctx,
      BB, 
      zoom = 90,
      offsetX,
      offsetY,
      WIDTH, 
      HEIGHT,
      dragok = false,
      startX = 0,
      startY = 0,
      meshSegments = [], 
      pointSize = 10,
      pointFill = '#00ccff',
      anchorFill = '#cccccc', 
      skeletons_state = true, 
      points_state = true, 
      canvasPadding = 400, 
      LUTsteps = 2, 
      themesh = [], 
      tempCurves = [];

  export let mkAspect = "mk360";
  
  let mkAspects = [{name: "mk360", width: 1920, height: 1080}, {name: "mkpro", width: 1920, height: 1200}]
  let aspect = [mkAspects[0]]

  let meshResolutions = [{columns: 6, rows: 4}]
  let meshResolution = meshResolutions[0]

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
	
  /*
  let steps = 100;
  let LUT = curve.getLUT(steps);
  */

  function drawCurve(ctx, curve, offset) {
    ctx.strokeStyle = "#00ccff";
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
    if(skeletons_state==true){
      var pts = curve.points;
      ctx.strokeStyle = "#cccccc";
      drawLine(ctx, pts[0], pts[1], offset);
      if (pts.length === 3) drawLine(ctx, pts[1], pts[2], offset);
      else drawLine(ctx, pts[2], pts[3], offset);
      ctx.strokeStyle = "#00ccff";
      // if(!nocoords) drawPoints(ctx, pts, offset);
    }
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

  function drawCircle(ctx, p, r, offset, fill) {
    offset = offset || { x:0, y:0 };
    var ox = offset.x;
    var oy = offset.y;
    // ctx.strokeStyle = "#ff0000";
    ctx.beginPath();
    ctx.arc(p.x + ox, p.y + oy, r, 0, 2*Math.PI);
    ctx.strokeStyle = '#003300';
    if(p.isDragging){
      ctx.arc(p.x + ox, p.y + oy, r*2, 0, 2*Math.PI);
    }
    ctx.fillStyle = p.fill || fill;
    ctx.fill();
    
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

  function drawSegments(){
    themesh = []
    meshSegments.map((segment, u) => {
      tempCurves  = []
      segment.segmentPoints.map((point, i)=>{ 
        if(u%2==0){
          if(i<(segment.segmentPoints.length-1)){
            if(i%2 == 0){
              let curvePoints = [point,segment.segmentPoints[i+1],segment.segmentPoints[i+2]]
              let curve = new Bezier(getSegmentPoints(curvePoints));
              drawSkeleton(ctx,curve)
              drawCurve(ctx,curve)
              let LUT = curve.getLUT(LUTsteps);
              LUT.map((fpoint,f)=>{
                if(i>0&&f==0){

                }else{
                  let tempx = (fpoint.x-(canvasPadding/2)).toFixed(8)/aspect[0].width
                  let tempy = (fpoint.y-(canvasPadding/2)).toFixed(8)/aspect[0].height
                  // themesh.push(tempx+", "+tempy)
                }
              })
              if(points_state){
                let offset = { x:0, y:0 };
                drawCircle(ctx, point, pointSize, offset, anchorFill)
                drawCircle(ctx, segment.segmentPoints[i+1], pointSize, offset, anchorFill)
                drawCircle(ctx, segment.segmentPoints[i+2], pointSize, offset, anchorFill)
              }
            }
          }
          if(u<(meshSegments.length-1)){
            let point_1 = meshSegments[u].segmentPoints[i]
            let point_2 = meshSegments[u+1].segmentPoints[i]
            let point_3 = meshSegments[u+2].segmentPoints[i]
            let hCurve = new Bezier(getSegmentPoints([point_1,point_2,point_3]));
            let tempLUT = hCurve.getLUT(LUTsteps);
            tempCurves.push(tempLUT)
            if(i%2 == 0){
              drawSkeleton(ctx,hCurve)
              drawCurve(ctx,hCurve)
              if(points_state){
                let offset = { x:0, y:0 };
                drawCircle(ctx, point_2, pointSize, offset, anchorFill)
              }
            }
          }
          
        }
      })
      
      if(tempCurves.length){
        for(let h=0;h<tempCurves[0].length;h++){
          if(u>0&&h==0){

          }else{
            tempCurves.map((pointArray,r)=>{  
              if(r%2==0){
                if(r<tempCurves.length-1){
                  let point_1 = pointArray[h]
                  let point_2 = tempCurves[r+1][h]
                  let point_3 = tempCurves[r+2][h]
                  let curvePoints = [point_1,point_2,point_3]
                  let curve = new Bezier(getSegmentPoints(curvePoints));
                  let LUT = curve.getLUT(LUTsteps);
                  LUT.map((fpoint,f)=>{
                    // console.log(f, fpoint)
                    if(r>0&&f==0){

                    }else{
                      let tempx = (fpoint.x-(canvasPadding/2)).toFixed(8)/aspect[0].width
                      let tempy = (fpoint.y-(canvasPadding/2)).toFixed(8)/aspect[0].height
                      themesh.push(tempx+", "+tempy)
                    }
                  })
                }
              }
            })
          }
        }
      }
      
      
    })
  }

  function draw() {
    clear();
    ctx.fillStyle = "#ffffff";
    rect(0, 0, WIDTH, HEIGHT);
    drawSegments();
  }

  function relativePoint(ex,ey) {
    let xPc = (ex)/BB.width;
    let yPc = (ey)/BB.height;
    let mx = (aspect[0].width+canvasPadding)*xPc;
    let my = (aspect[0].height+canvasPadding)*yPc;
    return { mx, my }
  }

  function myDown(e) {
    // tell the browser we're handling this mouse event
    e.preventDefault();
    e.stopPropagation();
    // get the current mouse position
    let tempx = parseInt(e.clientX - offsetX);
    var tempy = parseInt(e.clientY - offsetY);
    let {mx,my} = relativePoint(tempx,tempy);
    // test each rect to see if mouse is inside
    dragok = false;
    meshSegments.map((segment, u) => {
      for (var i = 0; i < segment.segmentPoints.length; i++) {
        var r = segment.segmentPoints[i];
        if (mx > (r.x-pointSize) && mx < (r.x+pointSize) && my > (r.y-pointSize) && my < (r.y+pointSize)) {
            // if yes, set that segmentPoints isDragging=true
            dragok = true;
            r.isDragging = true;
        }
      }
    })
    
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
    if (!e.shiftKey) {
      meshSegments.map((segment, u) => {
        for (var i = 0; i < segment.segmentPoints.length; i++) {
          segment.segmentPoints[i].isDragging = false;
        }
      })
    }draw()
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
        let {mx,my} = relativePoint(tempx,tempy);
        // calculate the distance the mouse has moved
        // since the last mousemove
        var dx = mx - startX;
        var dy = my - startY;
        // move each segmentPoint that isDragging
        // by the distance the mouse has moved
        // since the last mousemove
        meshSegments.map((segment, u) => {
          for (var i = 0; i < segment.segmentPoints.length; i++) {
              var r = segment.segmentPoints[i];
              if (r.isDragging) {
                  r.x += dx;
                  r.y += dy;
              }
          }
        })
        // redraw the scene with the new rect positions
        draw();
        // reset the starting mouse position for the next mousemove
        startX = mx;
        startY = my;
    }
  }

  function initMeshSegments(){
    meshSegments = []
    for (var x = 0, i = 0; i <= meshResolution.columns; x+=(aspect[0].width/meshResolution.columns), i++) {
      let tempSegment = {segmentPoints: []}
      for (var y = 0, j=0; j <= meshResolution.rows; y+=(aspect[0].height/meshResolution.rows), j++) {     
        let fill = pointFill
        if(j%2==1||i%2==1){
          fill = anchorFill
        }
        if(j%2==1&&i%2==1){
          fill = '#ffffff'
        }
        tempSegment.segmentPoints.push({
          x: x+(canvasPadding/2),
          y: y+(canvasPadding/2),
          width: pointSize*2,
          height: pointSize*2,
          fill: fill,
          isDragging: false
        })
      }
      meshSegments.push(tempSegment)
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
      initMeshSegments()
      draw()
    },1000)
    
	})

  function toggleSkeletons() {
    skeletons_state = !skeletons_state;
    points_state = !points_state;
    draw();
  }
  function reset() {
    initMeshSegments()
    draw();
  }

  function renderMesh(){
    // console.log(JSON.stringify(themesh))
    console.log(themesh)
    console.log("colums",(((meshResolutions[0].columns/2)*(LUTsteps-1))+(meshResolutions[0].columns/2)+1))
    console.log("rows",(((meshResolutions[0].rows/2)*(LUTsteps-1))+(meshResolutions[0].rows/2)+1))
  }

</script>

<div class="mesh_container">
  <div class="options">
    <button on:click={toggleSkeletons}>
      {#if !skeletons_state}
        <p>Show skeletons</p>
      {:else}
        <p>Hide skeletons</p>
      {/if}
    </button>
    <button on:click={reset}>
      Reset
    </button>
    <button on:click={renderMesh}>
      Render
    </button>
  </div>
  <canvas
    bind:this={canvas}
    width={aspect[0].width+canvasPadding}
    height={aspect[0].height+canvasPadding}
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
    display: flex;
    flex-direction: column;
    .options {
      display: flex;
      flex-direction: row;
      padding-bottom: 30px;
    }
  }
	canvas {
    border: 2px solid #f00;
    width: var(--zoom);
		background-color: #dfdfdf;
	}
</style>