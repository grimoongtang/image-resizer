<script lang="ts">
	import { resolve } from '$app/paths';
	import { onDestroy } from 'svelte';

	type OutputFormat = 'jpg' | 'png' | 'webp';

	const MAX_FILE_SIZE = 20 * 1024 * 1024;
	const ACCEPTED_MIME_TYPES = new Set([
		'image/jpeg',
		'image/png',
		'image/webp',
		'image/heic',
		'image/heif'
	]);
	const ACCEPTED_EXTENSIONS = new Set(['jpg', 'jpeg', 'png', 'webp', 'heic', 'heif']);
	const PRESETS = [
		{ name: '세로형', width: 1080, height: 1920 },
		{ name: '가로형', width: 1920, height: 1080 }
	];
	const SCALES = [25, 50, 75, 100];
	const FORMATS: OutputFormat[] = ['jpg', 'png', 'webp'];
	const MIN_ZOOM = 0.25;
	const MAX_ZOOM = 5;

	let uploadInput = $state<HTMLInputElement>();
	let previewCanvas = $state<HTMLDivElement>();
	let previewImage = $state<HTMLImageElement>();
	let selectedFile = $state<File | null>(null);
	let previewUrl = $state('');
	let originalWidth = $state(0);
	let originalHeight = $state(0);
	let outputWidth = $state(0);
	let outputHeight = $state(0);
	let widthDraft = $state('');
	let heightDraft = $state('');
	let ratioLocked = $state(true);
	let selectedScale = $state<number | null>(50);
	let outputFormat = $state<OutputFormat>('jpg');
	let errorMessage = $state('');
	let isDragging = $state(false);
	let isExporting = $state(false);
	let zoom = $state(1);
	let panX = $state(0);
	let panY = $state(0);
	let isPanning = $state(false);
	let pointerStartX = 0;
	let pointerStartY = 0;
	let panStartX = 0;
	let panStartY = 0;

	const originalRatio = $derived(originalWidth > 0 ? originalWidth / originalHeight : 1);
	const ratioLabel = $derived.by(() => {
		let a = originalWidth;
		let b = originalHeight;
		while (b) [a, b] = [b, a % b];
		return a ? `${originalWidth / a}:${originalHeight / a}` : '1:1';
	});
	const outputSizeLabel = $derived(`${outputWidth || 0} × ${outputHeight || 0} px`);

	onDestroy(() => {
		if (previewUrl) URL.revokeObjectURL(previewUrl);
	});

	function formatBytes(bytes: number): string {
		if (bytes < 1024 * 1024) return `${Math.max(1, Math.round(bytes / 1024))} KB`;
		return `${(bytes / (1024 * 1024)).toFixed(1)} MB`;
	}

	function resetSize() {
		if (!originalWidth || !originalHeight) return;
		outputWidth = Math.max(1, Math.round(originalWidth / 2));
		outputHeight = Math.max(1, Math.round(originalHeight / 2));
		widthDraft = String(outputWidth);
		heightDraft = String(outputHeight);
		ratioLocked = true;
		selectedScale = 50;
		outputFormat = 'jpg';
		errorMessage = '';
	}

	function resetPreview() {
		zoom = 1;
		panX = 0;
		panY = 0;
		isPanning = false;
	}

	function resetToUpload() {
		if (previewUrl) URL.revokeObjectURL(previewUrl);
		previewUrl = '';
		selectedFile = null;
		originalWidth = 0;
		originalHeight = 0;
		outputWidth = 0;
		outputHeight = 0;
		widthDraft = '';
		heightDraft = '';
		errorMessage = '';
		isDragging = false;
		resetPreview();
		if (uploadInput) uploadInput.value = '';
	}

	function setZoom(nextZoom: number, anchorX = 0, anchorY = 0) {
		const next = Math.max(MIN_ZOOM, Math.min(MAX_ZOOM, nextZoom));
		if (next === zoom) return;
		const scale = next / zoom;
		panX = anchorX - (anchorX - panX) * scale;
		panY = anchorY - (anchorY - panY) * scale;
		zoom = next;
	}

	function handleWheelZoom(event: WheelEvent) {
		event.preventDefault();
		if (!previewCanvas) return;
		const bounds = previewCanvas.getBoundingClientRect();
		const anchorX = event.clientX - bounds.left - bounds.width / 2;
		const anchorY = event.clientY - bounds.top - bounds.height / 2;
		setZoom(zoom * Math.exp(-event.deltaY * 0.0015), anchorX, anchorY);
	}

	function wheelZoom(node: HTMLDivElement) {
		node.addEventListener('wheel', handleWheelZoom, { passive: false });
		return { destroy: () => node.removeEventListener('wheel', handleWheelZoom) };
	}

	function startPan(event: PointerEvent) {
		if (event.button !== 0 || (event.target as HTMLElement).closest('button')) return;
		isPanning = true;
		pointerStartX = event.clientX;
		pointerStartY = event.clientY;
		panStartX = panX;
		panStartY = panY;
		(event.currentTarget as HTMLDivElement).setPointerCapture(event.pointerId);
	}

	function movePan(event: PointerEvent) {
		if (!isPanning) return;
		panX = panStartX + event.clientX - pointerStartX;
		panY = panStartY + event.clientY - pointerStartY;
	}

	function stopPan() {
		isPanning = false;
	}

	function loadImage(url: string): Promise<{ width: number; height: number }> {
		return new Promise((resolve, reject) => {
			const image = new Image();
			image.onload = () => resolve({ width: image.naturalWidth, height: image.naturalHeight });
			image.onerror = () => reject(new Error('이 브라우저에서 이미지를 열 수 없어요.'));
			image.src = url;
		});
	}

	async function receiveFile(file?: File) {
		if (!file) return;
		errorMessage = '';

		const extension = file.name.split('.').pop()?.toLowerCase() ?? '';
		if (!ACCEPTED_MIME_TYPES.has(file.type) && !ACCEPTED_EXTENSIONS.has(extension)) {
			errorMessage = 'JPG, PNG, WEBP, HEIC 파일만 선택할 수 있어요.';
			return;
		}
		if (file.size > MAX_FILE_SIZE) {
			errorMessage = '파일 크기는 20MB 이하로 선택해 주세요.';
			return;
		}

		const nextUrl = URL.createObjectURL(file);
		try {
			const dimensions = await loadImage(nextUrl);
			if (previewUrl) URL.revokeObjectURL(previewUrl);
			previewUrl = nextUrl;
			selectedFile = file;
			originalWidth = dimensions.width;
			originalHeight = dimensions.height;
			resetPreview();
			resetSize();
		} catch {
			URL.revokeObjectURL(nextUrl);
			errorMessage = extension === 'heic' || extension === 'heif'
				? '이 브라우저는 HEIC 미리보기를 지원하지 않아요. JPG나 PNG로 변환한 뒤 다시 선택해 주세요.'
				: '이미지를 열 수 없어요. 다른 파일을 선택해 주세요.';
		}
	}

	function handleFileChange(event: Event) {
		const input = event.currentTarget as HTMLInputElement;
		void receiveFile(input.files?.[0]);
		input.value = '';
	}

	function handleDrop(event: DragEvent) {
		event.preventDefault();
		isDragging = false;
		void receiveFile(event.dataTransfer?.files[0]);
	}

	function chooseScale(scale: number) {
		selectedScale = scale;
		outputWidth = Math.max(1, Math.round((originalWidth * scale) / 100));
		outputHeight = Math.max(1, Math.round((originalHeight * scale) / 100));
		widthDraft = String(outputWidth);
		heightDraft = String(outputHeight);
		ratioLocked = true;
		errorMessage = '';
	}

	function normalizeDimension(value: string): string {
		return value.replace(/\D/g, '').replace(/^0+/, '');
	}

	function setWidthDraft(value: string) {
		widthDraft = normalizeDimension(value);
		outputWidth = Number(widthDraft) || 0;
		selectedScale = null;
	}

	function commitWidthDraft() {
		if (ratioLocked && outputWidth > 0) {
			outputHeight = Math.max(1, Math.round(outputWidth / originalRatio));
			heightDraft = String(outputHeight);
		}
	}

	function setHeightDraft(value: string) {
		heightDraft = normalizeDimension(value);
		outputHeight = Number(heightDraft) || 0;
		selectedScale = null;
	}

	function commitHeightDraft() {
		if (ratioLocked && outputHeight > 0) {
			outputWidth = Math.max(1, Math.round(outputHeight * originalRatio));
			widthDraft = String(outputWidth);
		}
	}

	function choosePreset(width: number, height: number) {
		selectedScale = null;
		ratioLocked = false;
		outputWidth = width;
		outputHeight = height;
		widthDraft = String(width);
		heightDraft = String(height);
		errorMessage = '';
	}

	function toggleRatio() {
		ratioLocked = !ratioLocked;
		if (ratioLocked && outputWidth > 0) {
			outputHeight = Math.max(1, Math.round(outputWidth / originalRatio));
			heightDraft = String(outputHeight);
			selectedScale = null;
		}
	}

	function chooseFiles() {
		uploadInput?.click();
	}

	async function downloadImage() {
		if (!previewUrl || !outputWidth || !outputHeight || isExporting) return;
		isExporting = true;
		errorMessage = '';

		try {
			const image = new Image();
			image.src = previewUrl;
			await image.decode();

			const canvas = document.createElement('canvas');
			canvas.width = outputWidth;
			canvas.height = outputHeight;
			const context = canvas.getContext('2d');
			if (!context) throw new Error('이미지를 변환할 수 없어요.');

			if (outputFormat === 'jpg') {
				context.fillStyle = '#ffffff';
				context.fillRect(0, 0, outputWidth, outputHeight);
			}
			context.drawImage(image, 0, 0, outputWidth, outputHeight);

			const mimeType = outputFormat === 'jpg' ? 'image/jpeg' : `image/${outputFormat}`;
			const blob = await new Promise<Blob>((resolve, reject) => {
				canvas.toBlob(
					(result) => (result ? resolve(result) : reject(new Error('파일을 만들지 못했어요.'))),
					mimeType,
					0.92
				);
			});

			const downloadUrl = URL.createObjectURL(blob);
			const anchor = document.createElement('a');
			const baseName = selectedFile?.name.replace(/\.[^.]+$/, '') || 'image';
			anchor.href = downloadUrl;
			anchor.download = `${baseName}_resized.${outputFormat}`;
			anchor.click();
			window.setTimeout(() => URL.revokeObjectURL(downloadUrl), 1000);
		} catch (error) {
			errorMessage = error instanceof Error ? error.message : '다운로드에 실패했어요.';
		} finally {
			isExporting = false;
		}
	}
</script>

<svelte:head>
	<title>이미지 크기조절 | 그리뭉</title>
	<meta name="description" content="사진을 올리고 원하는 크기로 바꿔보세요." />
	<link rel="preconnect" href="https://fonts.googleapis.com" />
	<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin="anonymous" />
	<link
		href="https://fonts.googleapis.com/css2?family=Jua&family=Noto+Sans+KR:wght@400;500;700&display=swap"
		rel="stylesheet"
	/>
</svelte:head>

<div class="page-shell">
	<header class="site-header">
		<div class="header-inner">
			<a class="brand" href={resolve('/')} onclick={resetToUpload}>이미지 크기조절</a>
			<span class="byline">by 그리뭉</span>
		</div>
	</header>

	<main>
		{#if selectedFile}
			<section class="editor-workspace" aria-label="이미지 크기 조절 작업 영역">
				<div class="editor-grid">
					<section class="preview-card" aria-label="이미지 미리보기">
						<div class="file-toolbar">
							<div class="file-summary">
								<span class="file-icon"><img src="/icons/image.svg" alt="" /></span>
								<span class="file-copy">
									<strong>{selectedFile.name}</strong>
									<span>{selectedFile.name.split('.').pop()?.toUpperCase()} &nbsp;·&nbsp; {formatBytes(selectedFile.size)}</span>
								</span>
							</div>
							<button class="replace-button" type="button" onclick={chooseFiles}>
								<img src="/icons/replace.svg" alt="" />
								<span>이미지 바꾸기</span>
							</button>
						</div>

						<div
							bind:this={previewCanvas}
							use:wheelZoom
							class="preview-canvas"
							class:dragging={isDragging}
							class:panning={isPanning}
							onpointerdown={startPan}
							onpointermove={movePan}
							onpointerup={stopPan}
						onpointercancel={stopPan}
						onlostpointercapture={stopPan}
						ondragenter={(event) => { event.preventDefault(); isDragging = true; }}
						ondragover={(event) => event.preventDefault()}
						ondragleave={(event) => { if (event.currentTarget === event.target) isDragging = false; }}
						ondrop={handleDrop}
							role="region"
							aria-label="이미지 미리보기. 마우스 휠로 확대·축소하고 드래그해서 이동할 수 있어요."
						>
							<img
								bind:this={previewImage}
								class="preview-image"
								src={previewUrl}
								alt={`업로드한 이미지 ${selectedFile.name}`}
								draggable="false"
								style:transform={`translate(${panX}px, ${panY}px) scale(${zoom})`}
							/>
							<div class="zoom-controls" aria-label="미리보기 확대·축소">
								<button type="button" aria-label="축소" title="축소" onclick={() => setZoom(zoom / 1.25)}><span aria-hidden="true">−</span></button>
								<output aria-live="polite">{Math.round(zoom * 100)}%</output>
								<button type="button" aria-label="확대" title="확대" onclick={() => setZoom(zoom * 1.25)}><span aria-hidden="true">+</span></button>
								<button class="fit-button" type="button" onclick={resetPreview}>맞춤</button>
							</div>
						</div>

						<div class="image-metadata">
							<div class="metadata-item">
								<span>원본 크기</span>
								<strong>{originalWidth} × {originalHeight} px</strong>
							</div>
							<div class="metadata-item output-metadata">
								<span>변경할 크기</span>
								<strong>{outputWidth} × {outputHeight} px</strong>
							</div>
							<button class="reset-button" type="button" onclick={resetSize}>
								<img src="/icons/reset.svg" alt="" />
								<span>초기화</span>
							</button>
						</div>
					</section>

					<aside class="settings-card" aria-labelledby="settings-title">
						<h1 id="settings-title">크기 설정</h1>
						<div class="section-divider"></div>

						<div class="dimension-labels"><label for="width-input">가로</label><label for="height-input">세로</label></div>
						<div class="dimension-fields">
						<div class="dimension-field"><input id="width-input" type="text" inputmode="numeric" pattern="[0-9]*" placeholder="0" value={widthDraft} oninput={(event) => setWidthDraft(event.currentTarget.value)} onblur={commitWidthDraft} /><span>px</span></div>
						<div class="dimension-field"><input id="height-input" type="text" inputmode="numeric" pattern="[0-9]*" placeholder="0" value={heightDraft} oninput={(event) => setHeightDraft(event.currentTarget.value)} onblur={commitHeightDraft} /><span>px</span></div>
						</div>

						<div class="ratio-row">
							<span class="ratio-label"><img src="/icons/ratio.svg" alt="" />원본 비율 유지</span>
							<span class="ratio-value">{ratioLabel}</span>
							<button class="switch" class:enabled={ratioLocked} type="button" role="switch" aria-checked={ratioLocked} aria-label="원본 비율 유지" onclick={toggleRatio}><span></span></button>
						</div>

						<div class="control-section scale-section">
							<h2>비율로 조절하기</h2>
							<div class="scale-options">
								{#each SCALES as scale (scale)}
									<button class:active={selectedScale === scale} type="button" onclick={() => chooseScale(scale)}>{scale}%</button>
								{/each}
								<button class:active={selectedScale === null} type="button" onclick={() => (selectedScale = null)}>직접 입력</button>
							</div>
						</div>

						<div class="control-section presets-section">
							<h2>자주 쓰는 크기</h2>
							<div class="preset-options">
								{#each PRESETS as preset (preset.name)}
									<button type="button" onclick={() => choosePreset(preset.width, preset.height)}>
										<strong>{preset.name}</strong><span>{preset.width} × {preset.height}</span>
									</button>
								{/each}
							</div>
						</div>

						<div class="control-section format-section">
							<h2>저장할 파일 형식</h2>
							<div class="format-options">
								{#each FORMATS as format (format)}
									<button class:active={outputFormat === format} type="button" onclick={() => (outputFormat = format)}>{format.toUpperCase()}</button>
								{/each}
							</div>
						</div>

						<div class="download-summary"><span>저장 크기</span><strong>{outputSizeLabel}</strong></div>
						<button class="download-button" type="button" disabled={isExporting || outputWidth < 1 || outputHeight < 1} onclick={downloadImage}>
							<img src="/icons/download.svg" alt="" />
							<span>{isExporting ? '이미지 변환 중…' : '이미지 다운로드'}</span>
						</button>
					</aside>
				</div>
				{#if errorMessage}<p class="editor-error" role="alert">{errorMessage}</p>{/if}
				<input bind:this={uploadInput} class="visually-hidden" type="file" accept=".jpg,.jpeg,.png,.webp,.heic,.heif,image/jpeg,image/png,image/webp,image/heic,image/heif" onchange={handleFileChange} />
			</section>
		{:else}
			<section class="upload-hero" aria-labelledby="page-title">
				<div class="hero-inner">
					<h1 id="page-title">이미지 크기조절</h1>
					<p class="subtitle">사진을 올리고, 원하는 크기로 바꿔보세요.</p>
					<div
						class:dragging={isDragging}
						class="drop-zone"
						onclick={chooseFiles}
						onkeydown={(event) => event.key === 'Enter' && chooseFiles()}
						ondragenter={(event) => { event.preventDefault(); isDragging = true; }}
						ondragover={(event) => event.preventDefault()}
						ondragleave={(event) => { if (event.currentTarget === event.target) isDragging = false; }}
						ondrop={handleDrop}
						role="button"
						tabindex="0"
						aria-label="이미지 선택 또는 드래그 앤 드롭"
					>
						<p class="drop-title">이미지를 여기에 놓아주세요</p>
						<p class="drop-hint">또는 아래 버튼으로 파일을 선택하세요</p>
						<button class="upload-button" type="button" onclick={(event) => { event.stopPropagation(); chooseFiles(); }}>
							<img src="/upload.svg" alt="" width="20" height="20" /><span>이미지 선택하기</span>
						</button>
						<p class="file-types">JPG · PNG · WEBP&nbsp; / &nbsp;최대 20MB</p>
					</div>
					{#if errorMessage}<p class="upload-feedback error" role="alert">{errorMessage}</p>{/if}
					<p class="reassurance">회원가입 없이 무료로 이용할 수 있어요</p>
					<input bind:this={uploadInput} class="visually-hidden" type="file" accept=".jpg,.jpeg,.png,.webp,.heic,.heif,image/jpeg,image/png,image/webp,image/heic,image/heif" onchange={handleFileChange} />
				</div>
			</section>
		{/if}

		<aside class="ad-slot" aria-label="광고 영역">
			<div class="ad-inner"><span class="ad-label">광고</span><div class="ad-placeholder">Google AdSense · 970 × 90</div></div>
		</aside>
	</main>

	<footer class="site-footer">
		<div class="footer-inner">
			<a class="footer-brand" href={resolve('/')} onclick={resetToUpload}>이미지 크기조절</a>
			<nav class="footer-links" aria-label="푸터">
				<span class="footer-link-text">개인정보처리방침</span><span class="footer-link-text">이용약관</span><span>© 2026 그리뭉</span>
			</nav>
		</div>
	</footer>
</div>

<style>
	:global(*) { box-sizing: border-box; }
	:global(html) { min-width: 320px; background: #fff; }
	:global(body) { margin: 0; color: var(--color-brand-ink); font-family: 'Noto Sans KR', sans-serif; -webkit-font-smoothing: antialiased; }
	:global(button), :global(input) { font: inherit; }
	:global(button) { color: inherit; }
	.page-shell { min-height: 100vh; display: flex; flex-direction: column; }
	.site-header { height: 72px; flex: 0 0 72px; background: #fff; }
	.header-inner, .footer-inner { width: min(1280px, calc(100% - 64px)); height: 100%; margin: 0 auto; display: flex; align-items: center; justify-content: space-between; }
	.header-inner { border-bottom: 1px solid var(--color-brand-border); }
	.brand, .footer-brand { color: var(--color-brand-ink); font-family: Jua, sans-serif; text-decoration: none; }
	.brand { font-size: 30px; line-height: 42px; }
	.byline { color: var(--color-brand-muted); font-size: 14px; font-weight: 500; }
	h1, h2, p { margin-top: 0; }
	.upload-hero { min-height: 480px; padding: 38px 24px 28px; background: var(--color-brand-canvas); text-align: center; }
	.hero-inner { max-width: 880px; margin: 0 auto; }
	.upload-hero h1 { margin: 0; font-family: Jua, sans-serif; font-size: 44px; font-weight: 400; line-height: 62px; }
	.subtitle { margin: 8px 0 0; color: var(--color-brand-muted); font-size: 16px; line-height: 24px; }
	.drop-zone { min-height: 236px; margin-top: 32px; padding: 28px 20px 18px; border: 1px dashed var(--color-brand-border-strong); border-radius: 16px; background: #fff; cursor: pointer; transition: background-color 140ms ease, border-color 140ms ease; }
	.drop-zone:focus-visible { outline: 3px solid #80c4e8; outline-offset: 3px; }
	.drop-zone.dragging { border-color: var(--color-brand-primary); background: var(--color-brand-drop-hover); }
	.drop-title { margin: 0; font-size: 22px; font-weight: 700; line-height: 33px; }
	.drop-hint { margin: 4px 0 0; color: var(--color-brand-muted); font-size: 14px; line-height: 21px; }
	.upload-button { min-width: 208px; height: 48px; margin-top: 21px; padding: 0 22px; border: 0; border-radius: 14px; background: var(--color-brand-primary); color: white; display: inline-flex; align-items: center; justify-content: center; gap: 9px; font-size: 16px; font-weight: 700; cursor: pointer; }
	.upload-button:hover, .download-button:hover:not(:disabled) { background: var(--color-brand-primary-hover); }
	.file-types { margin: 17px 0 0; color: var(--color-brand-muted); font-size: 12px; line-height: 18px; }
	.visually-hidden { position: absolute; width: 1px; height: 1px; padding: 0; margin: -1px; overflow: hidden; clip: rect(0, 0, 0, 0); white-space: nowrap; clip-path: inset(50%); }
	.upload-feedback, .editor-error { margin: 12px 0 0; color: var(--color-brand-error); font-size: 13px; line-height: 20px; }
	.reassurance { margin: 23px 0 0; color: var(--color-brand-muted-strong); font-size: 13px; line-height: 20px; }
	.editor-workspace { min-height: 736px; padding: 24px max(32px, calc((100% - 1280px) / 2)) 32px; background: var(--color-brand-canvas); }
	.editor-grid { width: 100%; max-width: 1280px; margin: 0 auto; display: grid; grid-template-columns: minmax(0, 2fr) minmax(360px, 1.019fr); gap: 24px; align-items: stretch; }
	.preview-card, .settings-card { min-width: 0; height: 680px; border: 1px solid var(--color-brand-border); border-radius: 20px; background: #fff; }
	.preview-card { padding: 18px 10px 0; display: flex; flex-direction: column; }
	.file-toolbar { min-height: 36px; margin: 0 12px 20px; display: flex; align-items: center; justify-content: space-between; gap: 12px; }
	.file-summary { min-width: 0; display: flex; align-items: center; gap: 12px; }
	.file-icon { width: 36px; height: 36px; flex: 0 0 36px; border-radius: 9px; background: var(--color-brand-soft); display: grid; place-items: center; }
	.file-icon img { width: 20px; height: 20px; }
	.file-copy { min-width: 0; display: grid; gap: 1px; }
	.file-copy strong { overflow: hidden; color: var(--color-brand-ink); font-size: 14px; font-weight: 700; text-overflow: ellipsis; white-space: nowrap; }
	.file-copy span { color: var(--color-brand-muted); font-size: 12px; }
	.replace-button, .reset-button { border: 1px solid var(--color-brand-border); border-radius: 8px; background: #fff; display: inline-flex; align-items: center; justify-content: center; gap: 8px; cursor: pointer; }
	.replace-button { height: 36px; min-width: 155px; padding: 0 10px; font-size: 12px; font-weight: 500; }
	.replace-button img { width: 18px; height: 18px; }
	.preview-canvas { position: relative; min-height: 0; flex: 1; margin: 0; overflow: hidden; border-radius: 10px; background-color: var(--color-brand-checker-dark); background-image: linear-gradient(45deg, var(--color-brand-checker-light) 25%, transparent 25%, transparent 75%, var(--color-brand-checker-light) 75%), linear-gradient(45deg, var(--color-brand-checker-light) 25%, transparent 25%, transparent 75%, var(--color-brand-checker-light) 75%); background-position: 0 0, 20px 20px; background-size: 40px 40px; display: grid; place-items: center; touch-action: none; cursor: grab; }
	.preview-canvas.dragging { outline: 2px dashed var(--color-brand-primary); outline-offset: -6px; }
	.preview-canvas.panning { cursor: grabbing; }
	.preview-image { display: block; max-width: calc(100% - 32px); max-height: calc(100% - 32px); object-fit: contain; object-position: center; border-radius: 4px; box-shadow: 0 6px 14px rgb(26 58 79 / 16%); transform-origin: center; user-select: none; -webkit-user-drag: none; }
	.zoom-controls { position: absolute; top: 12px; right: 12px; z-index: 1; min-height: 40px; padding: 4px; border: 1px solid rgb(223 235 243 / 92%); border-radius: 10px; background: rgb(255 255 255 / 96%); display: flex; align-items: center; gap: 2px; box-shadow: 0 2px 8px rgb(36 62 82 / 10%); }
	.zoom-controls button { width: 32px; height: 30px; padding: 0; border: 0; border-radius: 6px; background: transparent; color: var(--color-brand-ink); display: grid; place-items: center; font-size: 20px; line-height: 1; cursor: pointer; }
	.zoom-controls button:hover { background: var(--color-brand-soft); }
	.zoom-controls output { min-width: 44px; color: var(--color-brand-muted); font-size: 11px; text-align: center; font-variant-numeric: tabular-nums; }
	.zoom-controls .fit-button { width: auto; min-width: 44px; margin-left: 2px; border-left: 1px solid var(--color-brand-border); border-radius: 0 6px 6px 0; color: var(--color-brand-primary); font-size: 11px; font-weight: 700; }
	.image-metadata { min-height: 84px; padding: 18px 12px 0; display: flex; align-items: flex-start; }
	.metadata-item { min-width: 135px; padding-right: 16px; display: grid; gap: 6px; }
	.metadata-item + .metadata-item { min-width: 185px; padding-left: 16px; border-left: 1px solid var(--color-brand-border); }
	.metadata-item span { color: var(--color-brand-subtle); font-size: 12px; font-weight: 700; }
	.metadata-item strong { color: var(--color-brand-ink); font-size: 14px; font-weight: 500; }
	.output-metadata strong { color: var(--color-brand-primary); }
	.reset-button { min-height: 30px; margin: 0 3px 0 auto; padding: 0 5px; border-color: transparent; color: var(--color-brand-muted); font-size: 13px; }
	.reset-button img { width: 17px; height: 17px; }
	.settings-card { padding: 16px 23px 22px; }
	.settings-card h1 { margin: 0; font-size: 20px; font-weight: 700; line-height: 30px; }
	.section-divider { height: 1px; margin: 12px 0 14px; background: var(--color-brand-border); }
	.dimension-labels, .dimension-fields { display: grid; grid-template-columns: 1fr 1fr; gap: 12px; }
	.dimension-labels { margin-bottom: 6px; }
	.dimension-labels label, .control-section h2 { color: var(--color-brand-ink); font-size: 13px; font-weight: 500; }
	.dimension-field { height: 48px; min-width: 0; padding: 0 12px; border: 1px solid var(--color-brand-field-border); border-radius: 10px; display: flex; align-items: center; gap: 6px; }
	.dimension-field:focus-within { border-color: var(--color-brand-primary); box-shadow: 0 0 0 2px rgb(21 119 175 / 12%); }
	.dimension-field input { width: 100%; min-width: 0; padding: 0; border: 0; outline: 0; color: var(--color-brand-ink); font-size: 14px; font-weight: 500; background: transparent; }
	.dimension-field input::-webkit-inner-spin-button, .dimension-field input::-webkit-outer-spin-button { margin: 0; appearance: none; }
	.dimension-field input[type='number'] { appearance: textfield; }
	.dimension-field span { color: var(--color-brand-subtle); font-size: 12px; }
	.ratio-row { min-height: 40px; display: flex; align-items: center; gap: 8px; }
	.ratio-label { margin-right: auto; display: inline-flex; align-items: center; gap: 8px; color: var(--color-brand-ink); font-size: 13px; font-weight: 500; }
	.ratio-label img { width: 18px; height: 18px; }
	.ratio-value { color: var(--color-brand-subtle); font-size: 12px; }
	.switch { width: 32px; height: 18px; padding: 2px; border: 0; border-radius: 999px; background: var(--color-brand-field-border); cursor: pointer; }
	.switch span { width: 14px; height: 14px; border-radius: 50%; background: #fff; display: block; transition: transform 140ms ease; }
	.switch.enabled { background: var(--color-brand-primary); }
	.switch.enabled span { transform: translateX(14px); }
	.control-section h2 { margin: 0 0 8px; line-height: 20px; }
	.scale-section { margin-top: 3px; }
	.scale-options { height: 38px; padding: 3px; border-radius: 8px; background: var(--color-brand-checker-dark); display: grid; grid-template-columns: repeat(5, 1fr); gap: 3px; }
	.scale-options button, .format-options button { padding: 0 4px; border: 0; border-radius: 6px; background: transparent; color: var(--color-brand-muted); font-size: 11px; cursor: pointer; }
	.scale-options button.active { border: 1px solid #cde7f6; background: #fff; color: var(--color-brand-primary); font-weight: 700; }
	.presets-section { margin-top: 16px; padding-bottom: 14px; border-bottom: 1px solid var(--color-brand-border); }
	.preset-options { display: grid; grid-template-columns: 1fr 1fr; gap: 8px; }
	.preset-options button { min-height: 36px; border: 1px solid var(--color-brand-border); border-radius: 8px; background: white; color: var(--color-brand-muted); display: grid; place-content: center; gap: 0; font-size: 11px; cursor: pointer; }
	.preset-options button strong { font-weight: 500; }
	.preset-options button span { font-size: 10px; }
	.preset-options button:hover, .replace-button:hover, .reset-button:hover { border-color: var(--color-brand-border-strong); background: var(--color-brand-hover); }
	.format-section { margin-top: 12px; }
	.format-options { height: 38px; padding: 3px; border: 1px solid var(--color-brand-border); border-radius: 8px; display: grid; grid-template-columns: repeat(3, 1fr); gap: 4px; }
	.format-options button { font-weight: 700; }
	.format-options button.active { background: var(--color-brand-soft); color: var(--color-brand-primary); }
	.download-summary { margin: 16px 0 12px; display: flex; align-items: center; justify-content: space-between; }
	.download-summary span { color: var(--color-brand-muted); font-size: 12px; }
	.download-summary strong { color: var(--color-brand-ink); font-size: 14px; }
	.download-button { width: 100%; height: 48px; border: 0; border-radius: 10px; background: var(--color-brand-primary); color: #fff; display: inline-flex; align-items: center; justify-content: center; gap: 8px; font-size: 14px; font-weight: 700; cursor: pointer; }
	.download-button img { width: 20px; height: 20px; }
	.download-button:disabled { opacity: 0.65; cursor: wait; }
	.editor-error { max-width: 1280px; margin: 8px auto 0; text-align: center; }
	.ad-slot { min-height: 158px; padding: 18px 24px 20px; background: #fff; }
	.ad-inner { width: min(970px, 100%); margin: 0 auto; }
	.ad-label { color: var(--color-brand-muted-strong); font-size: 11px; line-height: 17px; }
	.ad-placeholder { min-height: 90px; margin-top: 11px; border: 1px solid var(--color-brand-ad-border); border-radius: 8px; background: var(--color-brand-ad-background); color: var(--color-brand-muted-strong); display: grid; place-items: center; font-size: 14px; }
	.site-footer { min-height: 100px; background: #fff; }
	.footer-inner { border-top: 1px solid var(--color-brand-border); }
	.footer-brand { font-size: 18px; line-height: 27px; }
	.footer-links { display: flex; align-items: center; gap: 24px; color: var(--color-brand-muted); font-size: 12px; font-weight: 500; }
	.footer-link-text { color: inherit; }
	@media (max-width: 980px) {
		.editor-workspace { padding: 24px 24px 32px; }
		.editor-grid { grid-template-columns: minmax(0, 1.6fr) minmax(340px, 1fr); gap: 16px; }
		.preview-card { height: 620px; }
		.settings-card { height: auto; min-height: 620px; padding-inline: 18px; }
		.file-toolbar { margin-inline: 4px; }
		.metadata-item { min-width: 115px; }
		.metadata-item + .metadata-item { min-width: 155px; }
	}
	@media (max-width: 720px) {
		.site-header { height: 64px; flex-basis: 64px; }
		.header-inner, .footer-inner { width: calc(100% - 40px); }
		.brand { font-size: 23px; }
		.upload-hero { min-height: 460px; padding: 34px 20px 28px; }
		.upload-hero h1 { font-size: 36px; line-height: 50px; }
		.subtitle { font-size: 14px; }
		.drop-zone { min-height: 220px; margin-top: 30px; padding: 27px 14px 16px; }
		.drop-title { font-size: 19px; }
		.drop-hint { font-size: 13px; }
		.editor-workspace { padding: 16px 16px 24px; }
		.editor-grid { grid-template-columns: minmax(0, 1fr); gap: 14px; }
		.preview-card { height: min(590px, 125vw); min-height: 440px; padding-inline: 8px; }
		.file-toolbar { margin-inline: 4px; }
		.replace-button { min-width: 40px; width: 40px; padding: 0; }
		.replace-button span { display: none; }
		.image-metadata { padding-inline: 4px; }
		.metadata-item { min-width: 0; flex: 1; }
		.metadata-item + .metadata-item { min-width: 0; }
		.metadata-item strong { font-size: 12px; }
		.reset-button { font-size: 12px; }
		.settings-card { height: auto; min-height: 0; padding: 18px; }
		.ad-slot { min-height: 134px; padding: 16px 20px 20px; }
		.ad-placeholder { min-height: 76px; }
		.site-footer { min-height: 118px; }
		.footer-inner { padding: 20px 0; align-items: flex-start; flex-direction: column; gap: 12px; }
		.footer-links { flex-wrap: wrap; gap: 10px 18px; }
	}
</style>
