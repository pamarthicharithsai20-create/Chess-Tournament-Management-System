<script>
	let players = [];
	let tournaments = [];
	let matches = [];

	let playerName = '';
	let tournamentName = '';
	let activeTournamentId = null;

	let editingPlayerId = null;
	let editingPlayerName = '';
	let editingTournamentName = '';

	let nextPlayerId = 1;
	let nextTournamentId = 1;
	let nextMatchId = 1;

	$: activeTournament = tournaments.find((tournament) => tournament.id === activeTournamentId);

	$: tournamentPlayers = activeTournament
		? players.filter((player) => activeTournament.playerIds.includes(player.id))
		: [];

	$: availablePlayers = activeTournament
		? players.filter((player) => !activeTournament.playerIds.includes(player.id))
		: [];

	$: tournamentMatches = activeTournament
		? matches.filter((match) => match.tournamentId === activeTournament.id)
		: [];

	$: rankings = activeTournament
		? [...tournamentPlayers].sort((a, b) => {
				const aScore = activeTournament.scores[a.id] || { wins: 0, losses: 0 };
				const bScore = activeTournament.scores[b.id] || { wins: 0, losses: 0 };

				return (
					bScore.wins - aScore.wins ||
					aScore.losses - bScore.losses ||
					a.name.localeCompare(b.name)
				);
			})
		: [];

	function addPlayer() {
		const name = playerName.trim();
		if (!name) return;

		players = [...players, { id: nextPlayerId++, name }];
		playerName = '';
	}

	function savePlayer(playerId) {
		const name = editingPlayerName.trim();
		if (!name) return;

		players = players.map((player) => (player.id === playerId ? { ...player, name } : player));

		matches = matches.map((match) => ({
			...match,
			playerA: match.playerA.id === playerId ? { ...match.playerA, name } : match.playerA,
			playerB: match.playerB.id === playerId ? { ...match.playerB, name } : match.playerB
		}));

		editingPlayerId = null;
		editingPlayerName = '';
	}

	function deletePlayer(playerId) {
		players = players.filter((player) => player.id !== playerId);

		tournaments = tournaments.map((tournament) => {
			const updatedScores = { ...tournament.scores };
			delete updatedScores[playerId];

			return {
				...tournament,
				playerIds: tournament.playerIds.filter((id) => id !== playerId),
				scores: updatedScores
			};
		});

		matches = matches.filter(
			(match) => match.playerA.id !== playerId && match.playerB.id !== playerId
		);
	}

	function createTournament() {
		const name = tournamentName.trim();
		if (!name) return;

		const tournament = {
			id: nextTournamentId++,
			name,
			playerIds: [],
			scores: {}
		};

		tournaments = [...tournaments, tournament];
		activeTournamentId = tournament.id;
		tournamentName = '';
	}

	function saveTournament() {
		const name = editingTournamentName.trim();
		if (!name || !activeTournament) return;

		tournaments = tournaments.map((tournament) =>
			tournament.id === activeTournamentId ? { ...tournament, name } : tournament
		);

		editingTournamentName = '';
	}

	function deleteTournament(tournamentId) {
		tournaments = tournaments.filter((tournament) => tournament.id !== tournamentId);
		matches = matches.filter((match) => match.tournamentId !== tournamentId);

		if (activeTournamentId === tournamentId) {
			activeTournamentId = null;
		}
	}

	function addToTournament(playerId) {
		if (!activeTournament || activeTournament.playerIds.includes(playerId)) return;

		tournaments = tournaments.map((tournament) => {
			if (tournament.id !== activeTournamentId) return tournament;

			return {
				...tournament,
				playerIds: [...tournament.playerIds, playerId],
				scores: {
					...tournament.scores,
					[playerId]: { wins: 0, losses: 0 }
				}
			};
		});
	}

	function removeFromTournament(playerId) {
		tournaments = tournaments.map((tournament) => {
			if (tournament.id !== activeTournamentId) return tournament;

			const updatedScores = { ...tournament.scores };
			delete updatedScores[playerId];

			return {
				...tournament,
				playerIds: tournament.playerIds.filter((id) => id !== playerId),
				scores: updatedScores
			};
		});

		matches = matches.filter(
			(match) =>
				match.tournamentId !== activeTournamentId ||
				(match.playerA.id !== playerId && match.playerB.id !== playerId)
		);
	}

	function shuffle(items) {
		const shuffled = [...items];

		for (let i = shuffled.length - 1; i > 0; i--) {
			const randomIndex = Math.floor(Math.random() * (i + 1));
			[shuffled[i], shuffled[randomIndex]] = [shuffled[randomIndex], shuffled[i]];
		}

		return shuffled;
	}

	function generateMatches() {
		if (!activeTournament || tournamentPlayers.length < 2) return;

		const shuffledPlayers = shuffle(tournamentPlayers);
		const newMatches = [];

		for (let i = 0; i + 1 < shuffledPlayers.length; i += 2) {
			newMatches.push({
				id: nextMatchId++,
				tournamentId: activeTournamentId,
				playerA: shuffledPlayers[i],
				playerB: shuffledPlayers[i + 1]
			});
		}

		matches = [...matches.filter((match) => match.tournamentId !== activeTournamentId), ...newMatches];
	}

	function recordWinner(match, winnerId) {
		const loserId = match.playerA.id === winnerId ? match.playerB.id : match.playerA.id;

		tournaments = tournaments.map((tournament) => {
			if (tournament.id !== activeTournamentId) return tournament;

			return {
				...tournament,
				scores: {
					...tournament.scores,
					[winnerId]: {
						...tournament.scores[winnerId],
						wins: tournament.scores[winnerId].wins + 1
					},
					[loserId]: {
						...tournament.scores[loserId],
						losses: tournament.scores[loserId].losses + 1
					}
				}
			};
		});

		matches = matches.filter((item) => item.id !== match.id);
	}

	function scoreFor(playerId) {
		return activeTournament?.scores[playerId] || { wins: 0, losses: 0 };
	}
</script>

<svelte:head>
	<title>Chess Tournament Management System</title>
</svelte:head>

<main class="app">
	<!-- Header -->
	<header class="hero">
		<div class="hero-badge">Tournament Organizer</div>
		<h1>Chess Tournament<br />Management System</h1>
		<p>Create players and tournaments, generate pairings, and manually record every match winner.</p>
	</header>

	<!-- Player and Tournament Management -->
	<section class="setup-grid">
		<article class="panel">
			<div class="panel-heading">
				<div>
					<span>Player Management</span>
					<h2>Registered Players</h2>
				</div>
				<div class="number">{players.length}</div>
			</div>

			<form class="add-form" on:submit|preventDefault={addPlayer}>
				<input bind:value={playerName} placeholder="Enter player name" />
				<button type="submit">Add Player</button>
			</form>

			<div class="item-list">
				{#if players.length === 0}
					<p class="empty-message">No players added.</p>
				{:else}
					{#each players as player (player.id)}
						<div class="list-card">
							{#if editingPlayerId === player.id}
								<input bind:value={editingPlayerName} />
								<div class="button-group">
									<button on:click={() => savePlayer(player.id)}>Save</button>
									<button class="button-light" on:click={() => (editingPlayerId = null)}>Cancel</button>
								</div>
							{:else}
								<strong>{player.name}</strong>
								<div class="button-group">
									<button
										class="button-light"
										on:click={() => {
											editingPlayerId = player.id;
											editingPlayerName = player.name;
										}}>Edit</button
									>
									<button class="button-danger" on:click={() => deletePlayer(player.id)}>Delete</button>
								</div>
							{/if}
						</div>
					{/each}
				{/if}
			</div>
		</article>

		<article class="panel">
			<div class="panel-heading">
				<div>
					<span>Tournament Management</span>
					<h2>Your Tournaments</h2>
				</div>
				<div class="number">{tournaments.length}</div>
			</div>

			<form class="add-form" on:submit|preventDefault={createTournament}>
				<input bind:value={tournamentName} placeholder="Tournament name" />
				<button type="submit">Create Tournament</button>
			</form>

			<div class="item-list">
				{#if tournaments.length === 0}
					<p class="empty-message">No tournaments created.</p>
				{:else}
					{#each tournaments as tournament (tournament.id)}
						<div class:active-item={tournament.id === activeTournamentId} class="list-card tournament-item">
							<div>
								<strong>{tournament.name}</strong>
								<small>{tournament.playerIds.length} player(s)</small>
							</div>
							<div class="button-group">
								<button on:click={() => (activeTournamentId = tournament.id)}>Open</button>
								<button class="button-danger" on:click={() => deleteTournament(tournament.id)}>Delete</button>
							</div>
						</div>
					{/each}
				{/if}
			</div>
		</article>
	</section>

	{#if activeTournament}
		<!-- Active Tournament -->
		<section class="panel tournament-panel">
			<div class="active-header">
				<div>
					<span>Active Tournament</span>

					{#if editingTournamentName !== ''}
						<div class="name-editor">
							<input bind:value={editingTournamentName} />
							<button on:click={saveTournament}>Save</button>
							<button class="button-light" on:click={() => (editingTournamentName = '')}>Cancel</button>
						</div>
					{:else}
						<h2>{activeTournament.name}</h2>
					{/if}
				</div>

				{#if editingTournamentName === ''}
					<button
						class="button-light"
						on:click={() => (editingTournamentName = activeTournament.name)}
					>
						Edit Tournament
					</button>
				{/if}
			</div>

			<div class="tournament-grid">
				<div>
					<h3>Players in Tournament</h3>

					{#if availablePlayers.length > 0}
						<div class="available-players">
							{#each availablePlayers as player (player.id)}
								<button class="button-light" on:click={() => addToTournament(player.id)}>
									+ {player.name}
								</button>
							{/each}
						</div>
					{/if}

					<div class="item-list">
						{#if tournamentPlayers.length === 0}
							<p class="empty-message">Add players to this tournament.</p>
						{:else}
							{#each tournamentPlayers as player (player.id)}
								<div class="list-card">
									<div>
										<strong>{player.name}</strong>
										<small>{scoreFor(player.id).wins} wins · {scoreFor(player.id).losses} losses</small>
									</div>
									<button class="button-danger" on:click={() => removeFromTournament(player.id)}>
										Remove
									</button>
								</div>
							{/each}
						{/if}
					</div>
				</div>

				<div>
					<h3>Match System</h3>
					<button
						class="generate-button"
						on:click={generateMatches}
						disabled={tournamentPlayers.length < 2}
					>
						Generate Random Matches
					</button>

					<div class="match-list">
						{#if tournamentMatches.length === 0}
							<p class="empty-message">No matches generated.</p>
						{:else}
							{#each tournamentMatches as match (match.id)}
								<div class="match-card">
									<div class="versus">
										<strong>{match.playerA.name}</strong>
										<span>VS</span>
										<strong>{match.playerB.name}</strong>
									</div>
									<p>Choose the winner after the match.</p>
									<div class="winner-buttons">
										<button on:click={() => recordWinner(match, match.playerA.id)}>
											{match.playerA.name} Wins
										</button>
										<button class="button-light" on:click={() => recordWinner(match, match.playerB.id)}>
											{match.playerB.name} Wins
										</button>
									</div>
								</div>
							{/each}
						{/if}
					</div>
				</div>
			</div>
		</section>

		<!-- Rankings -->
		<section class="panel rankings-panel">
			<div class="panel-heading">
				<div>
					<span>Tournament Results</span>
					<h2>Final Rankings</h2>
				</div>
			</div>

			<div class="podium">
				{#each [0, 1, 2] as place}
					<div class="podium-card">
						<span>{place + 1}{place === 0 ? 'st' : place === 1 ? 'nd' : 'rd'} Place</span>
						<strong>{rankings[place]?.name || '—'}</strong>
					</div>
				{/each}
			</div>

			<div class="table-wrapper">
				<table>
					<thead>
						<tr>
							<th>Rank</th>
							<th>Player Name</th>
							<th>Wins</th>
							<th>Losses</th>
						</tr>
					</thead>
					<tbody>
						{#if rankings.length === 0}
							<tr>
								<td colspan="4" class="empty-message">No players added to this tournament.</td>
							</tr>
						{:else}
							{#each rankings as player, index (player.id)}
								<tr>
									<td>{index + 1}</td>
									<td><strong>{player.name}</strong></td>
									<td>{scoreFor(player.id).wins}</td>
									<td>{scoreFor(player.id).losses}</td>
								</tr>
							{/each}
						{/if}
					</tbody>
				</table>
			</div>
		</section>
	{/if}
</main>

<style>
	@import url('https://fonts.googleapis.com/css2?family=Poppins:wght@400;500;600;700&display=swap');

	:global(*) {
		box-sizing: border-box;
	}

	:global(body) {
		margin: 0;
		background: #f4f7fb;
		color: #0f172a !important;
		font-family: 'Poppins', sans-serif;
	}

	:global(button),
	:global(input) {
		font: inherit;
	}

	.app {
		width: min(1240px, calc(100% - 32px));
		margin: 0 auto;
		padding: 48px 0 72px;
	}

	.hero {
		padding: 18px 0 34px;
		border-bottom: 1px solid #d8e1ee;
		text-align: center;
	}

	.hero-badge,
	.panel-heading span,
	.active-header span {
		color: #2563eb !important;
		font-size: 0.76rem;
		font-weight: 700;
		letter-spacing: 0.1em;
		text-transform: uppercase;
	}

	h1,
	h2,
	h3,
	strong,
	th,
	td {
		color: #0f172a !important;
	}

	h1 {
		margin: 12px 0;
		font-size: clamp(2rem, 5vw, 3.3rem);
		line-height: 1.15;
		letter-spacing: -0.04em;
	}

	h2 {
		margin: 4px 0 0;
		font-size: 1.35rem;
	}

	h3 {
		margin: 0 0 14px;
		font-size: 1.05rem;
	}

	.hero p {
		margin: 0;
		color: #64748b !important;
		font-size: 1rem;
	}

	.setup-grid,
	.tournament-grid {
		display: grid;
		grid-template-columns: repeat(2, minmax(0, 1fr));
		gap: 22px;
	}

	.setup-grid {
		margin-top: 30px;
	}

	.panel {
		background: #ffffff;
		border: 1px solid #e1e9f3;
		border-radius: 18px;
		box-shadow: 0 12px 28px rgba(15, 23, 42, 0.06);
		padding: 24px;
	}

	.panel-heading,
	.active-header {
		display: flex;
		align-items: flex-start;
		justify-content: space-between;
		gap: 16px;
		margin-bottom: 20px;
	}

	.number {
		display: grid;
		place-items: center;
		width: 34px;
		height: 34px;
		border-radius: 50%;
		background: #e8f0ff;
		color: #2563eb;
		font-weight: 700;
	}

	.add-form,
	.name-editor {
		display: flex;
		gap: 10px;
	}

	input {
		width: 100%;
		min-width: 0;
		border: 1px solid #cbd7e6;
		border-radius: 10px;
		background: #ffffff;
		color: #0f172a !important;
		outline: none;
		padding: 11px 13px;
	}

	input::placeholder {
		color: #94a3b8;
	}

	input:focus {
		border-color: #2563eb;
		box-shadow: 0 0 0 3px rgba(37, 99, 235, 0.14);
	}

	button {
		border: 0;
		border-radius: 10px;
		background: #2563eb;
		color: #ffffff !important;
		cursor: pointer;
		font-size: 0.84rem;
		font-weight: 600;
		padding: 10px 14px;
		transition: background 0.2s ease, transform 0.2s ease;
		white-space: nowrap;
	}

	button:hover:not(:disabled) {
		background: #1d4ed8;
		transform: translateY(-1px);
	}

	button:disabled {
		cursor: not-allowed;
		opacity: 0.5;
	}

	.button-light {
		background: #edf3ff;
		color: #334155 !important;
	}

	.button-light:hover:not(:disabled) {
		background: #dce9fc;
	}

	.button-danger {
		background: #fff0f1;
		color: #dc2626 !important;
	}

	.button-danger:hover:not(:disabled) {
		background: #fee2e2;
	}

	.item-list,
	.match-list {
		display: grid;
		gap: 10px;
		margin-top: 16px;
	}

	.list-card {
		display: flex;
		align-items: center;
		justify-content: space-between;
		gap: 12px;
		border: 1px solid #e5edf6;
		border-radius: 12px;
		background: #ffffff;
		padding: 13px;
	}

	.list-card strong {
		display: block;
		font-size: 0.96rem;
	}

	.list-card small {
		display: block;
		color: #64748b;
		font-size: 0.78rem;
		margin-top: 3px;
	}

	.active-item {
		border-color: #93b4f9;
		background: #f7faff;
	}

	.button-group,
	.available-players,
	.winner-buttons {
		display: flex;
		flex-wrap: wrap;
		gap: 8px;
	}

	.empty-message {
		margin: 0;
		border: 1px dashed #cbd7e6;
		border-radius: 12px;
		color: #64748b !important;
		padding: 20px;
		text-align: center;
	}

	.tournament-panel,
	.rankings-panel {
		margin-top: 24px;
	}

	.available-players {
		margin-bottom: 14px;
	}

	.generate-button {
		width: 100%;
		padding: 13px;
	}

	.match-card {
		border: 1px solid #e4edf8;
		border-radius: 14px;
		background: #fbfdff;
		padding: 16px;
	}

	.versus {
		display: grid;
		grid-template-columns: 1fr auto 1fr;
		align-items: center;
		gap: 10px;
		text-align: center;
	}

	.versus span {
		color: #2563eb;
		font-size: 0.76rem;
		font-weight: 700;
	}

	.match-card p {
		margin: 10px 0 13px;
		color: #64748b !important;
		font-size: 0.8rem;
		text-align: center;
	}

	.winner-buttons {
		justify-content: center;
	}

	.podium {
		display: grid;
		grid-template-columns: repeat(3, 1fr);
		gap: 14px;
		margin-bottom: 24px;
	}

	.podium-card {
		border: 1px solid #dce7f8;
		border-radius: 14px;
		background: #f8fbff;
		padding: 18px;
		text-align: center;
	}

	.podium-card span {
		display: block;
		color: #2563eb !important;
		font-size: 0.78rem;
		font-weight: 700;
		text-transform: uppercase;
	}

	.podium-card strong {
		display: block;
		margin-top: 8px;
		font-size: 1.05rem;
	}

	.table-wrapper {
		overflow-x: auto;
	}

	table {
		width: 100%;
		border-collapse: collapse;
	}

	th,
	td {
		border-bottom: 1px solid #e8eef6;
		padding: 13px 11px;
		text-align: left;
	}

	th {
		color: #64748b !important;
		font-size: 0.73rem;
		letter-spacing: 0.06em;
		text-transform: uppercase;
	}

	td {
		font-size: 0.88rem;
	}

	@media (max-width: 760px) {
		.app {
			width: min(100% - 24px, 1240px);
			padding-top: 24px;
		}

		.setup-grid,
		.tournament-grid,
		.podium {
			grid-template-columns: 1fr;
		}

		.add-form,
		.name-editor,
		.active-header {
			flex-direction: column;
		}

		.add-form button,
		.name-editor button {
			width: 100%;
		}

		.panel {
			padding: 18px;
		}

		.list-card {
			align-items: flex-start;
			flex-direction: column;
		}

		.versus {
			grid-template-columns: 1fr;
		}
	}
</style>
