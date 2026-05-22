<script lang="ts">
	import { getColorSync, getPaletteSync, getSwatchesSync, createColor, type Color } from 'colorthief';
  import 'eyedropper-polyfill';
  import EyeDropperIcon from '$lib/assets/eyedropper-svgrepo-com.svg';

	let imageUrl = $state('');
	let isDragging = $state(false);
	let colorPalette = $state([]);
	let numOfColors = $state(6);

  let eyedropColor = $state(createColor(255, 255, 255, 1));
  let eyedropPicked = $state(false);
	let dominantColor = $state('');
	let semanticSwatches = $state({});

	let fileInput: HTMLInputElement;
	const semanticLabels = [
		'Vibrant',
		'Muted',
		'DarkVibrant',
		'DarkMuted',
		'LightVibrant',
		'LightMuted'
	];

	function handleDragOver(e: DragEvent): void {
		e.preventDefault();
		isDragging = true;
	}

	function handleDragLeave(): void {
		isDragging = false;
	}

	function processFile(file: File): void {
		if (file.type.startsWith('image/')) {
			const reader = new FileReader();
			reader.onload = (event) => {
				const result = event.target?.result;
				if (typeof result === 'string') {
					imageUrl = result;
					extractColors();
				}
			};
			reader.readAsDataURL(file);
		}
	}

	function handleDrop(e: DragEvent) {
		e.preventDefault();
		isDragging = false;

		const files = e.dataTransfer?.files;
		if (files && files.length > 0) {
			processFile(files[0]);
		}
	}

	function handleClick() {
		fileInput.click();
	}

	function handleKeyDown(e: KeyboardEvent) {
		if (e.key === 'Enter' || e.key === ' ') {
			e.preventDefault();
			handleClick();
		}
	}

	function handleFileInputChange(e: Event) {
		const input = e.target as HTMLInputElement;
		if (input.files && input.files.length > 0) {
			processFile(input.files[0]);
		}
	}

	// async function copyToClipboard(hex: string) {
	// 	console.log('Copying to clipboard:', hex);
	// 	if (!hex) {
	// 		console.warn('No hex value provided');
	// 		return;
	// 	}
	// 	try {
	// 		await navigator.clipboard.writeText(hex);
	// 		console.log(`Successfully copied ${hex} to clipboard`);
	// 	} catch (err) {
	// 		console.error('Failed to copy to clipboard:', err);
	// 	}
	// }

	function copyToClipboard(hex) {
		const copyText = document.createElement('input');
		copyText.value = hex;
		document.body.appendChild(copyText);

		copyText.select(); // Select the text field
		copyText.setSelectionRange(0, 99999); // For mobile devices

		document.execCommand('copy'); // execCommand is deprecated. // Execute the copy command
		// alert('Copied text!');
		document.body.removeChild(copyText);
	}

	function extractColors() {
		const img = new Image();
		img.crossOrigin = 'Anonymous';
		img.onload = () => {
			try {
				const palette = getPaletteSync(img, { colorCount: numOfColors, quality: 10 });
				const dominant = getColorSync(img);
				const semantic = getSwatchesSync(img);
				if (palette) {
					colorPalette = palette;
				}
				if (dominant) {
					dominantColor = dominant;
				}
				if (semantic) {
					semanticSwatches = semantic;
				}
			} catch (err) {
				console.error('Error extracting colors:', err);
			}
		};
		img.src = imageUrl;
	}

  function hexToRgbBitwise(hex) {
    // Convert hex to a number (0x notation)
    const num = parseInt(hex.replace(/^#/, ''), 16);

    return [(num >> 16) & 255, (num >> 8) & 255, num & 255];
  }

  function OpenEye() {
      const eyeDropper = new window.EyeDropper();

      eyeDropper.open()
      .then((colorSelectionResult) => {
          // Use the selected color information
          console.log('Selected color:', colorSelectionResult.sRGBHex);
          const rgb = hexToRgbBitwise(colorSelectionResult.sRGBHex);
          console.log('rgb', rgb)
          eyedropColor = createColor(rgb[0],rgb[1],rgb[2], 1);
          eyedropPicked = true;
        })
      .catch((error) => {
          console.error('Error:', error)
        })
    }

</script>

{#snippet clickableThing(content)}
	<div
		class="clickable"
		title="click to copy"
		onclick={() => copyToClipboard(content)}
		onkeydown={(e) => (e.key === 'Enter' || e.key === ' ') && copyToClipboard(content)}
		role="button"
		tabindex="0"
		aria-label="Copy dominant color {content} to clipboard"
	>
		{content}
	</div>
{/snippet}

{#snippet colorSwatch(colorObj, square)}
	<div class="color-item">
		<div class="color-swatch" class:square style="background-color: {colorObj.color.hex()};"></div>
		<div
			class="color-info"
			style="color: {colorObj.color.textColor}; --focus-color: {colorObj.color.textColor};"
		>
			<!-- <div class="color-hex">{colorObj.color.hex()}</div> -->
			{#if colorObj.label}
				<div class="color-label">{colorObj.label}</div>
			{/if}
			<div class="extra-info">
				{@render clickableThing(colorObj.color.hex())}
				{@render clickableThing(colorObj.color.css('rgb'))}
				{@render clickableThing(colorObj.color.css('hsl'))}
				{@render clickableThing(colorObj.color.css('oklch'))}
			</div>
		</div>
	</div>
{/snippet}

{#snippet proportionBar(palette)}
	<div class="proportion-bar">
		{#each palette as color, i (i)}
			<div
				class="proportion-bar-segment"
				style="flex:{color.proportion};background:{color.hex()}"
				title="{color.hex()} {Math.round(color.proportion * 100)}%"
			>
				<span class="proportion-bar-label" style="color:{color.textColor}">
					{Math.ceil(color.proportion.toFixed(2) * 100)}%
				</span>
			</div>
		{/each}
	</div>
{/snippet}

<div class="wrap">
	<div class="row">
		<h1>Extract Colors</h1>
		<h2>
			<a href="https://github.com/lokesh/color-thief">lokesh/color-thief</a> is doing all the work here
		</h2>
	</div>
	<div
		id="drop-zone"
		class="drop-zone"
		class:dragging={isDragging}
		ondragover={handleDragOver}
		ondragleave={handleDragLeave}
		ondrop={handleDrop}
		onclick={handleClick}
		onkeydown={handleKeyDown}
		role="button"
		aria-label="Drop zone for image upload or click to browse"
		tabindex="0"
	>
		<div class="drop-zone-label default-label">Drag an image here or click to browse</div>
		<div class="drop-zone-label dragging-label">Drop it!</div>
	</div>

	<input
		bind:this={fileInput}
		type="file"
		accept="image/*"
		onchange={handleFileInputChange}
		style="display: none;"
	/>

	<label for="numOfColors"
		>How many palette colors?
		<input type="number" id="numOfColors" min="2" max="20" bind:value={numOfColors} />
	</label>

	{#if imageUrl}
		<div class="result-container">
			<div class="image-container">
				<h2>Image</h2>
        <div class="eyedropwrap">
        <button onclick={OpenEye}><img class="eyedroppericon" src={EyeDropperIcon} alt="eyedropper icon"/></button>
        

        {#if eyedropPicked}
        {@render colorSwatch({ color: eyedropColor })}
        {/if}
        <!-- <div style="width:2rem, height:2rem;">{JSON.stringify(eyedropColor)}</div> -->
        </div>
				<img src={imageUrl} alt="Uploaded" />
				{@render proportionBar(colorPalette)}
			</div>

			{#if colorPalette.length > 0}
				<div class="palette-container">
					<h2>Dominant</h2>
					{@render colorSwatch({ color: dominantColor })}

					<h2>Semantic</h2>
					<div class="color-palette">
						{#each semanticLabels as label (label)}
							{#if semanticSwatches[label]?.color.hex()}
								{@render colorSwatch({
									color: semanticSwatches[label].color,
									label: semanticSwatches[label].role
								})}
							{/if}
						{/each}
					</div>

					<h2>Palette</h2>
					<div class="color-palette">
						{#each colorPalette as color (color.hex())}
							{@render colorSwatch({ color })}
						{/each}
					</div>
				</div>
			{/if}
		</div>
	{/if}
</div>

<style>
.eyedroppericon {
    width: 3rem;
  }
</style>
