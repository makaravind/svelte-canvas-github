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
				img.scaleToWidth(100);
				img.scaleToHeight(100);
				img.top = 10
				img.left = 50
				resolve(img);
			});
		});

		const cardWidth = 200;

		let nameText = new fabric.Text(card.name, {
			fontSize: 24,
			fontFamily: 'Arial',
			textAlign: 'center',
			left: 50,
			top: 110
		});

		nameText.set({
			left: (cardWidth - nameText.width!) / 2
		});

		let usernameText = new fabric.Text(card.username, {
			fontSize: 14,
			fontFamily: 'Arial',
			textAlign: 'center',
			fill: '#606060',
			left: 50,
			top: 140
		});

		usernameText.set({
			left: (cardWidth - usernameText.width!) / 2
		});

		let rect = new fabric.Rect({
			left: 0,
			top: 0,
			fill: '#fff',
			stroke: '#007bff',
			strokeWidth: 2,
			width: cardWidth,
			height: 165,
			rx: 10,
			ry: 10
		});

		let group = new fabric.Group([rect, avatarImg, nameText, usernameText], {
			left: 20, // Maintain a fixed horizontal margin
			top: 220 * cardCount, // Increase vertical position with each card
			centeredScaling: true // Center the group while scaling
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

		return group;
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

	function makeCanvasInteractive() {
		const onChange = (options: fType.IEvent) => {
			console.log("aravind:", 'called')
			options.target?.setCoords();
			canvas?.forEachObject(function(obj) {
				if (obj === options.target) return;
				obj.set('opacity' ,options.target?.intersectsWithObject(obj) ? 0.5 : 1);
			});
		}

		canvas?.on('object:moving', onChange)
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
		makeCanvasInteractive()
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
    /* Width */
    ::-webkit-scrollbar {
        width: 8px; /* Width of the scroll bar */
    }

    /* Track */
    ::-webkit-scrollbar-track {
        background: #f1f1f1; /* Color of the track */
    }

    /* Handle */
    ::-webkit-scrollbar-thumb {
        background: #007bff; /* Color of the handle */
        border-radius: 4px; /* Roundness of the handle */
    }

    /* Handle on hover */
    ::-webkit-scrollbar-thumb:hover {
        background: #555; /* Color of the handle on hover */
    }

	.container {
			width: 500px;
			height: 500px;
			overflow-y: auto;
			overflow-x: hidden;
	}

	.search {
      padding: 10px; /* Add some padding for spacing */
      border: 1px solid #ccc; /* Add a border */
      border-radius: 5px; /* Add rounded corners */
      font-size: 16px; /* Set font size */
			margin-bottom: 10px;
			width: 500px;
	}

  .search:focus {
      border-color: #007bff; /* Change border color when focused */
      outline: none; /* Remove default focus outline */
      box-shadow: 0 0 5px #007bff; /* Add a subtle box shadow when focused */
  }

	.heading {
      font-size: 36px; /* Set font size */
      font-weight: bold; /* Set font weight */
      color: #333; /* Set text color */
      margin-bottom: 20px; /* Add some space at the bottom */
	}

	.page-info {
			font-weight: 400;
	}
</style>



<div>
	<p  class="heading">Showing Mozilla Org members</p>
	<p>
		Page <span class="page-info">1</span>
	</p>
</div>

<input class="search" bind:value={searchVal} placeholder="search github ID" on:keyup={handleKeydown} />



<div class="container">
	<canvas id="c" width="500" height="500" ></canvas>
</div>