<script>
    import * as d3 from 'd3';
    import Lineplot from './Lineplot.svelte';
	import UsMap from './USMap.svelte';
    import UsMapInteractive from './USMapInteractive.svelte';
    let data = d3.ticks(-2, 2, 200).map(Math.sin);
    let mouseMoveChartOn = false

    let isMouseFiring = false
    function throttleMouseMove(e) {
        if (!isMouseFiring) {
            setTimeout(() => {
                onMousemove(e);
                isMouseFiring = false;
            }, 50);
        }
        isMouseFiring = true;
    }
    
    function onMousemove(event) {
          const [x, y] = d3.pointer(event);
      data = data.slice(-200).concat(Math.atan2(x, y));
    }

    
</script>

<div class="flex flex-row my-4 mx-4">
    <div class="text-4xl">
        SvelteKit + <br/> D3.js <br/> Testing
    </div>
    <div class="pb-24 flex flex-col justify-between w-full min-h-full">
        <div class="max-w-screen-2xl mx-auto w-full px-3 min-h-full">
            <div class="my-12">
                <div class="text-4xl text-center">
                    Testing Examples
                </div>
            </div>
            <hr class="my-12 h-0.5 border-t-0 bg-neutral-100 dark:bg-white/10" />
            <div class="mt-6">
                <div class="text-xl text-center my-6">
                    Mouse Move Reactive Chart
                </div>
                <div class="flex justify-center">
                    {#if mouseMoveChartOn == true}
                    <!-- svelte-ignore a11y-no-static-element-interactions -->
                    <div  on:mousemove={throttleMouseMove}>
                        <Lineplot data={data} />
                    </div>
                    <button class=btn on:click={() => mouseMoveChartOn = false}>
                        Done
                    </button>
                    {:else}
                    <button class="btn" on:click={() => mouseMoveChartOn = true}>
                        Start Mouse-move Chart Test
                    </button>
                    {/if}
                </div>
            </div>
            <hr class="my-12 h-0.5 border-t-0 bg-neutral-100 dark:bg-white/10" />
            <div class="mt-6">
                <div class="text-xl text-center my-6">
                    US SVG Chart
                </div>
                <div>
                    <UsMap/>
                </div>
            </div>
            <hr class="my-12 h-0.5 border-t-0 bg-neutral-100 dark:bg-white/10" />
            <div class="mt-6">
                <div class="text-xl text-center my-6">
                    US SVG Interactive
                </div>
                <div>
                    <UsMapInteractive/>
                </div>
            </div>
        </div>
    </div>
</div>