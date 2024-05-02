<!--<svelte:head>-->
<!--	<script src="https://cdnjs.cloudflare.com/ajax/libs/fabric.js/500/fabric.min.js"></script>-->
<!--</svelte:head>-->
<svelte:window on:keydown={saveImages}/>

<script lang="ts">
	import { onMount, tick } from 'svelte';
	import { fabric } from 'fabric';
	import { fabric as fType } from 'fabric';
	import pkg from 'file-saver';
	import { exportCardsAsImage, showImageInSidebar } from '$lib/utils/export';
	const {saveAs} = pkg;

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
	let canvasCards: fType.Group[]  = []
	let memberDetails : NameCard[]
	let page: number = 1;
	let searchVal = '';


	async function getNameCardRect(card: NameCard, cardCount: number) {
		const avatarImg = await new Promise<fType.Image>((resolve) => {
			fabric.Image.fromURL(card.avatar, (img) => {
				img.scaleToWidth(100);
				img.scaleToHeight(100);
				img.top = 10
				img.left = 50
				resolve(img);
			}, { crossOrigin: 'anonymous' });
		});

		avatarImg.set({ crossOrigin: 'anonymous' });


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

	function saveImages(event: KeyboardEvent) {
		if (event.ctrlKey && event.key === 's') {
			event.preventDefault();
			const image = exportCardsAsImage(canvas);
			if (image !== null) {
				showImageInSidebar(image);
			}
		}
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
			options.target?.setCoords();
			canvas?.forEachObject(function(obj) {
				if (obj === options.target) return;
				obj.set('opacity' ,options.target?.intersectsWithObject(obj) ? 0.5 : 1);
			});
		}

		canvas?.on('object:moving', onChange)
	}

	onMount(async () => {
		canvas = new fabric.Canvas('c', {
			enableRetinaScaling: true,
		});
		canvas.setHeight(0)
		await initContent()
	});

	async function updateCanvas(memberDetails: NameCard[]) {
		canvas?.clear();
		canvasCards = [];
		for (let i = 0; i < memberDetails.length; i++) {
			const member = memberDetails[i]
			const group = await getNameCardRect(member, i);
			updateCanvasHeight(i + 1)
			canvas?.add(group);
			canvasCards.push(group)
		}
		makeCanvasInteractive()
	}

	async function initContent() {
		memberDetails = await fetchMembers(1)
		await updateCanvas(memberDetails);
	}

	async function fetchMembers(page: number): Promise<NameCard[]> {
		console.log("aravind:", "Fetching members for page", page)
		const res = await fetch(`https://api.github.com/orgs/mozilla/members?page=${page}`)
		if(!res.ok) {
			memberDetails = []
			return [];
		}

		memberDetails = (await res.json() as unknown as GithubApiMemberResponse[]).map(r => ({
			name: r.login,
			profile: r.html_url,
			username: r.login,
			avatar: r.avatar_url
		}))

		return memberDetails;
	}

	async function handleNextPage() {
		page++
		memberDetails = await fetchMembers(page)
		await updateCanvas(memberDetails);
	}

	async function handlePrevPage() {
		if(page === 1) {
			return;
		}
		page--;
		memberDetails = await fetchMembers(page)
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
			color: #a9a9a9;
	}

	.search-container {
			display: flex;
			gap: 8px;
	}
    .pagination-btn {
        background-color: #007bff; /* Button background color */
        color: #fff; /* Text color */
        border: none; /* Remove border */
        padding: 8px 16px; /* Reduced padding */
        font-size: 14px; /* Reduced font size */
        border-radius: 5px; /* Add rounded corners */
        cursor: pointer; /* Change cursor to pointer on hover */
    }

    /* Styling for pagination buttons on hover */
    .pagination-btn:hover {
        background-color: #0056b3; /* Darker background color on hover */
    }

    /* Styling for pagination buttons on active/focus */
    .pagination-btn:focus,
    .pagination-btn:active {
        outline: none; /* Remove default focus outline */
        box-shadow: 0 0 5px rgba(0, 123, 255, 0.5); /* Add box shadow on focus/active */
    }

    #sidebar-container {
        display: flex;
        flex-direction: column;
        justify-content: center;
        align-items: center;
        width: 300px;
        background: #23acea;
        color: #fff;
        height: 100vh;
        position: fixed;
        top: 0;
        right: 0;
    }

    #sidebar {
        display: flex;
        flex-direction: column;
        justify-content: center;
        align-items: center;
        width: 100%;
        height: 100%;
    }
    .shortcut {
        display: flex;
        flex-direction: row;
        align-items: center;
        justify-content: space-between;
        font-family: 'Courier New', Courier, monospace;
        font-size: 14px;
        color: #a9a9a9;
        margin: 10px;
        padding: 0 10px;
        background-color: #242423;
        border-radius: 5px;
    }
</style>



<div>
	<p  class="heading">Showing Mozilla Org members</p>
</div>

<div class="search-container">
	<input class="search" bind:value={searchVal} placeholder="Search for Github ID" on:keyup={handleKeydown} />
	<button class="pagination-btn" on:click={handlePrevPage}>previous</button>
	<p class="page-info">
		Page {page}
	</p>
	<button class="pagination-btn" on:click={handleNextPage}>next</button>
</div>

<div class="container">
	<canvas id="c" width="500" height="500" ></canvas>
</div>

<div id="sidebar-container">
	<div class="shortcut">
		<strong>export : </strong>
		<p>alt+s</p>
	</div>
	<h3>Images</h3>
	<div id="sidebar">
	</div>
</div>