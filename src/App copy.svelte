<script>

  import { Bezier } from "bezier-js";
	import { onMount } from 'svelte';
  export let canvasHeight = ""
  let b = new Bezier(150,40 , 80,30 , 105,150);

  let mkAspects = [
    {name: "mk360", width: 1920, height: 1080},
    {name: "mkpro", width: 1920, height: 1200},
  ]

  let mkAspect = "mk360";
  
  let getAspect = (mkAspect) =>{
    let aspect = mkAspects.filter((aspect)=>{
      if(aspect.name == mkAspect){
        aspect.heightPercent = ( aspect.height * 100 ) / aspect.width
        return (aspect)
      }
    })
    return aspect
  }
	let canvas;

	onMount(() => {

    let aspect = getAspect(mkAspect)
    canvasHeight = `${aspect[0].heightPercent.toString()}vw`

		const ctx = canvas.getContext('2d');
		let frame;

		(function loop() {
			frame = requestAnimationFrame(loop);

			const imageData = ctx.getImageData(0, 0, canvas.width, canvas.height);

			for (let p = 0; p < imageData.data.length; p += 4) {
				const i = p / 4;
				const x = i % canvas.width;
				const y = i / canvas.height >>> 0;

				const t = window.performance.now();

				const r = 64 + (128 * x / canvas.width) + (64 * Math.sin(t / 1000));
				const g = 64 + (128 * y / canvas.height) + (64 * Math.cos(t / 1400));
				const b = 128;

				imageData.data[p + 0] = r;
				imageData.data[p + 1] = g;
				imageData.data[p + 2] = b;
				imageData.data[p + 3] = 255;
			}

			ctx.putImageData(imageData, 0, 0);
		}());

		return () => {
			cancelAnimationFrame(frame);
		};
	});
</script>

<canvas
	bind:this={canvas}
	width={32}
	height={32}
  style="--canvasHeight:{canvasHeight}"
></canvas>

<style>
	canvas {
		width: 100vw;
		height: var(--canvasHeight);
		background-color: #666;
		-webkit-mask: url(/svelte-logo-mask.svg) 50% 50% no-repeat;
		mask: url(/svelte-logo-mask.svg) 50% 50% no-repeat;
	}
</style>