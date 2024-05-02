<!--<svelte:head>-->
<!--	<script src="https://cdnjs.cloudflare.com/ajax/libs/fabric.js/500/fabric.min.js"></script>-->
<!--</svelte:head>-->

<script lang="ts">
	import { onMount, tick } from 'svelte';
	import { fabric } from 'fabric';
	import { fabric as fType } from 'fabric';

	interface NameCard {
		name: string;
		username: string;
		avatar: string;
		profile: string;
	}

	interface GithubApiMemberResponse {
		login: string;
		avatar_url: string;
		html_url: string;
	}

	let canvas: fType.Canvas | undefined;
	let memberDetails : NameCard[]
	let searchVal = '';


	async function getNameCardRect(card: NameCard, cardCount: number) {
		const avatarImg = await new Promise<fType.Image>((resolve, reject) => {
			fabric.Image.fromURL(card.avatar, (img) => {
				img.scaleToWidth(60);
				img.scaleToHeight(60);
				resolve(img);
			});
		});

		let nameText = new fabric.Text(card.name, {
			fontSize: 20,
			fontFamily: 'Arial',
			left: 70,
			top: 5
		});

		let usernameText = new fabric.Text(card.username, {
			fontSize: 16,
			fontFamily: 'Arial',
			left: 70,
			top: 30,
			fill: 'gray'
		});

		let profileText = new fabric.Text(card.profile, {
			fontSize: 14,
			fontFamily: 'Arial',
			left: 70,
			top: 50,
			width: 180,
			splitByGrapheme: true
		});

		let rect = new fabric.Rect({
			left: 0,
			top: 0,
			fill: '#fff',
			stroke: '#000',
			strokeWidth: 2,
			width: 300,
			height: 100
		});

		let group = new fabric.Group([rect, avatarImg, nameText, usernameText, profileText], {
			left: 20, // Maintain a fixed horizontal margin
			top: 120 * cardCount // Increase vertical position with each card
		});

		group.on('mousedblclick', () => {
			window.open(card.profile, '_blank');
		});

		group.on('mouseover', () => {
			rect.set({ stroke: 'blue' });
			canvas?.requestRenderAll();
		});

		group.on('mouseout', () => {
			rect.set({ stroke: '#000' });
			canvas?.requestRenderAll();
		});

		return group
	}

	async function handleKeydown(event: KeyboardEvent) {
		event.preventDefault();
		await tick();

		if(!searchVal || !searchVal.trim()) {
			await updateCanvas(memberDetails)
			return;
		}
		const viewMembers = memberDetails.filter(m => m.name.startsWith(searchVal))
		if(!viewMembers.length) {
			await updateCanvas(memberDetails)
			return;
		}
		await updateCanvas(viewMembers)
	}

	function updateCanvasHeight(cardCount: number) {
		const canvasHeight = cardCount * 120; // Update canvas height based on the number of cards
		canvas?.setHeight(canvasHeight);
	}

	onMount(() => {
		canvas = new fabric.Canvas('c');
		canvas.setHeight(0)
		init().then(() => console.log("aravind:", 'init'));
	});

	async function updateCanvas(memberDetails: NameCard[]) {
		canvas?.clear();
		for (let i = 0; i < memberDetails.length; i++) {
			const member = memberDetails[i]
			const group = await getNameCardRect(member, i);
			updateCanvasHeight(i + 1)
			canvas?.add(group);
		}
	}

	async function init() {
		const res = await fetch('https://api.github.com/orgs/mozilla/members?page=1')
		if(!res.ok) {
			memberDetails = []
			return;
		}

		memberDetails = (await res.json() as unknown as GithubApiMemberResponse[]).map(r => ({
			name: r.login,
			profile: r.html_url,
			username: r.login,
			avatar: r.avatar_url
		}))

		await updateCanvas(memberDetails);
	}
</script>

<style>
	.container {
			width: 500px;
			height: 500px;
			overflow-y: auto;
			scrollbar-width: none;
	}
</style>


<input bind:value={searchVal} placeholder="search github id" on:keyup={handleKeydown} />

<div class="container">
	<canvas id="c" width="500" height="500" ></canvas>
</div>