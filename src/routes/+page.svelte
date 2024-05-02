<!--<svelte:head>-->
<!--	<script src="https://cdnjs.cloudflare.com/ajax/libs/fabric.js/500/fabric.min.js"></script>-->
<!--</svelte:head>-->

<script lang="ts">
	import { onMount } from 'svelte';
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

	async function getNameCardRect(card: NameCard) {
		const avatarImg = await new Promise<fType.Image>((resolve) => {
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
			left: 100,
			top: 100
		});

		return group;
	}

	onMount(() => {
		canvas = new fabric.Canvas('c');
		init().then(() => console.log("aravind:", 'init'));
	});

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

		for (let i = 0; i < memberDetails.length; i++) {
			const member = memberDetails[i];
			const group = await getNameCardRect(member);
			canvas?.add(group);
		}
	}
</script>

<div class="MarkdownEditor">
	<canvas id="c" width="500" height="500" style="border:1px solid #000000;"></canvas>
</div>
