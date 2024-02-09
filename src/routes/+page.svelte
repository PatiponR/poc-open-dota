<script>
    import { onMount } from 'svelte';
    import { Button, Table, Input, Card } from 'flowbite-svelte';
    
    let steamID3 = '88223015';
    let playerStats = {};
    let playerTotals = [];
    let matchHistory = [];
    let matchDetails = null;
    let currentPage = 1;
    const rowsPerPage = 30;
    let paginatedMatches = [];
    let viewingMatchDetails = false;
    let heroes = [];

    async function fetchHeroes() {
        try {
            const response = await fetch('https://api.opendota.com/api/heroes');
            heroes = await response.json();
        } catch (error) {
            console.error('Error fetching heroes:', error);
        }
    }

    async function fetchData() {
        const accountId = steamID3;
        try {
            const [statsResponse, wlResponse, totalsResponse, matchesResponse] = await Promise.all([
                fetch(`https://api.opendota.com/api/players/${accountId}`),
                fetch(`https://api.opendota.com/api/players/${accountId}/wl`),
                fetch(`https://api.opendota.com/api/players/${accountId}/totals`),
                fetch(`https://api.opendota.com/api/players/${accountId}/matches`),
            ]);
            playerStats = await statsResponse.json();
            playerStats.winLose = await wlResponse.json();
            playerTotals = await totalsResponse.json();
            matchHistory = await matchesResponse.json();
            paginateMatches();
        } catch (error) {
            console.error("Error fetching data:", error);
        }
    }

    async function fetchMatchDetails(matchId) {
        try {
            const response = await fetch(`https://api.opendota.com/api/matches/${matchId}`);
            matchDetails = await response.json();
            viewingMatchDetails = true;
        } catch (error) {
            console.error("Error fetching match details:", error);
        }
    }

    function viewPlayerStats() {
        viewingMatchDetails = false;
    }

    function paginateMatches() {
        const start = (currentPage - 1) * rowsPerPage;
        const end = start + rowsPerPage;
        paginatedMatches = matchHistory.slice(start, end);
    }

    function changePage(newPage) {
        currentPage = newPage;
        paginateMatches();
    }

    function formatDuration(seconds) {
        const minutes = Math.floor(seconds / 60);
        const secondsRemaining = seconds % 60;
        return `${minutes}:${secondsRemaining < 10 ? '0' : ''}${secondsRemaining}`;
    }

    onMount(async () => {
        await fetchHeroes();
    });

    function getHeroName(heroId) {
        const hero = heroes.find(hero => hero.id === heroId);
        return hero ? hero.localized_name : 'Unknown';
    }
</script>

<div class="container mx-auto p-4">
    <div class="mb-6 flex justify-center">
        <Input type="text" bind:value={steamID3} placeholder="Enter SteamID3" class="mr-2 w-full max-w-xs" />
        <Button on:click={fetchData}>Confirm</Button>
    </div>

    {#if !viewingMatchDetails}
        {#if playerStats && playerStats.winLose}
            <Card class="w-full">
                <h2 class="text-lg font-semibold mb-4">Player Statistics</h2>
                <p>MMR Estimate: <span class="font-bold">{playerStats.mmr_estimate?.estimate || 'N/A'}</span></p>
                <p>Wins: <span class="font-bold">{playerStats.winLose.win}</span></p>
                <p>Losses: <span class="font-bold">{playerStats.winLose.lose}</span></p>
                {#each playerTotals as total}
                    <p>{total.field}: <span class="font-bold">{total.sum}</span></p>
                {/each}
            </Card>
        {/if}
    {:else}
        {#if matchDetails}
            <Card class="w-full">
                <h2 class="text-lg font-semibold mb-4">Match Details</h2>
                <p>Match ID: <span class="font-bold">{matchDetails.match_id}</span></p>
                <p>Duration: <span class="font-bold">{formatDuration(matchDetails.duration)}</span></p>
                <p>Winner: <span class="font-bold">{matchDetails.radiant_win ? 'Radiant' : 'Dire'}</span></p>
                <p>Game Mode: <span class="font-bold">{matchDetails.game_mode}</span></p>
                <p>Lobby Type: <span class="font-bold">{matchDetails.lobby_type}</span></p>
                <h3 class="font-semibold mt-4">Player Performances:</h3>
                <ul>
                    {#each matchDetails.players as player}
                        <li>Player ID: {player.account_id}, Hero: {getHeroName(player.hero_id)}, K/D/A: {player.kills}/{player.deaths}/{player.assists}, Last Hits: {player.last_hits}, Denies: {player.denies}, GPM: {player.gold_per_min}, XPM: {player.xp_per_min}</li>
                    {/each}
                </ul>
                <Button on:click={viewPlayerStats} class="mt-4">Back to Player Stats</Button>
            </Card>
        {/if}
    {/if}

    {#if paginatedMatches.length}
        <h2 class="font-bold text-lg mt-6">Match History</h2>
        <Table>
            <thead>
                <tr>
                    <th>Match ID</th>
                    <th>Hero</th>
                    <th>K/D/A</th>
                    <th>Result</th>
                </tr>
            </thead>
            <tbody>
                {#each paginatedMatches as match}
                    <tr class="cursor-pointer" on:click={() => fetchMatchDetails(match.match_id)}>
                        <td>{match.match_id}</td>
                        <td>{getHeroName(match.hero_id)}</td>
                        <td>{match.kills}/{match.deaths}/{match.assists}</td>
                        <td>{match.radiant_win ? 'Win' : 'Loss'}</td>
                    </tr>
                {/each}
            </tbody>
        </Table>
        <div class="flex justify-between mt-4">
            <Button disabled={currentPage <= 1} on:click={() => changePage(currentPage - 1)}>Previous</Button>
            <Button disabled={(currentPage * rowsPerPage) >= matchHistory.length} on:click={() => changePage(currentPage + 1)}>Next</Button>
        </div>
    {/if}
</div>
