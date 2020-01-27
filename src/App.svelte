<script>
	const YT_API_KEY = "AIzaSyDZuvLUXo6wzSMGiAOdFWNVXBjVh2MKPYE";

	let aliceChannelID = "UC4yuIyhAY1ODuYJux8d3VTg";
	let bobChannelID = "UCDE7MoDdJwleaIAd3qFJF1A";
	let isLoadingGapi = true;
	let isLoadingSubs = false;
	let subsLoadError;
	let bobSubs = new Set();
	let aliceSubs = new Set();
	let channelLinks = [];

	function gapiOnLoad() {
		gapi.load("client", loadClient);
	}

	function loadClient() {
		gapi.client.setApiKey(YT_API_KEY);
		return gapi.client
			.load("https://www.googleapis.com/discovery/v1/apis/youtube/v3/rest")
			.then(
				function() {
					console.log("GAPI client loaded for API");
					isLoadingGapi = false;
					onSubmit();
				},
				function(err) {
					console.error("Error loading GAPI client for API", err);
					isLoadingGapi = false;
				}
			);
	}

	async function onSubmit() {
		subsLoadError = null;
		isLoadingSubs = true;
		try {
			if (!(await lookupChannel(aliceChannelID))) {
				subsLoadError = "Alice has an invalid channel ID!";
				return;
			}
			if (!(await lookupChannel(bobChannelID))) {
				subsLoadError = "Bob has an invalid channel ID!";
				return;
			}
			await lookupSubs(aliceChannelID, aliceSubs);
			await lookupSubs(bobChannelID, bobSubs);
		} finally {
			isLoadingSubs = false;
		}
	}

	async function lookupChannel(channelID) {
		let result = await gapi.client.youtube.channels.list({
			part: "snippet",
			id: channelID
		});
		let data = JSON.parse(result.body);
		return data.items[0];
	}

	async function lookupSubs(channelID, resultSet) {
		resultSet.clear();
		let nextPageToken;
		do {
			let result = await gapi.client.youtube.subscriptions.list({
				part: "snippet,contentDetails",
				channelId: channelID,
				maxResults: 50,
				pageToken: nextPageToken
			});

			let data = JSON.parse(result.body);
			nextPageToken = data.nextPageToken;

			for (let channel of data.items) {
				let channelId = channel.snippet.resourceId.channelId;
				let channelLink = `'<a target="_blank" href='https://www.youtube.com/channel/${channelId}'>${channel.snippet.title}</a>'`;
				channelLinks[channelId] = channelLink;
				resultSet.add(channelId);
			}
		} while (nextPageToken);
	}

	function renderSubs(subs) {
		let links = [];
		for (let channelID of subs) {
			links.push(channelLinks[channelID]);
		}
		return `{${links.join(", ")}}`;
	}

	function union(a, b) {
		return new Set([...a, ...b]);
	}

	function intersection(a, b) {
		return new Set([...a].filter(x => b.has(x)));
	}

	function difference(a, b) {
		return new Set([...a].filter(x => !b.has(x)));
	}
</script>

<style>
	main {
		text-align: center;
	}

	input {
		width: 500px;
	}

	span {
		padding: 10px;
	}

	pre {
		font-size: 18pt;
	}
</style>

<svelte:head>
	<script src="https://apis.google.com/js/api.js" on:load={gapiOnLoad}>

	</script>
</svelte:head>
<main>
	<h2>Set operations on YouTube Subscriptions</h2>
	<hr />
	{#if isLoadingGapi}
		<div>Loading YouTube API...</div>
	{:else}
		<form on:submit={onSubmit} onsubmit="return false;">
			<div>
				<span>Alice's channel ID</span>
				<input autofocus bind:value={aliceChannelID} />
			</div>
			<br />
			<div>
				<span>Bob's channel ID</span>
				<input bind:value={bobChannelID} />
			</div>
			<br />
			<div>
				<button type="submit">Compute!</button>
			</div>
		</form>
		<hr />
		{#if isLoadingSubs}
			<span>Loading subscriptions...</span>
		{:else if subsLoadError}
			<span style="color: red;">{subsLoadError}</span>
		{:else}
			<pre>A</pre>
			<code>
				{@html renderSubs(aliceSubs)}
			</code>

			<hr />

			<pre>B</pre>
			<code>
				{@html renderSubs(bobSubs)}
			</code>

			<hr />

			<pre>A ∪ B</pre>
			<code>
				{@html renderSubs(union(aliceSubs, bobSubs))}
			</code>

			<hr />

			<pre>A ∩ B</pre>
			<code>
				{@html renderSubs(intersection(aliceSubs, bobSubs))}
			</code>

			<hr />

			<pre>A - B</pre>
			<code>
				{@html renderSubs(difference(aliceSubs, bobSubs))}
			</code>

			<hr />

			<pre>B - A</pre>
			<code>
				{@html renderSubs(difference(bobSubs, aliceSubs))}
			</code>
		{/if}
	{/if}
</main>
