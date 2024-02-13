<script>
    import { onMount } from 'svelte';
    import { Button, Card, Input, Table, TableBody, TableBodyCell, TableBodyRow, TableHead, TableHeadCell } from 'flowbite-svelte';
    import { Bar } from 'svelte-chartjs';
    import Chart from 'chart.js/auto';

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
    let playerHistogram = [];

    let histogramChartData = {
        labels: [],
        datasets: [
            {
                label: 'Games',
                data: [],
                backgroundColor: 'rgba(54, 162, 235, 0.2)',
                borderColor: 'rgba(54, 162, 235, 1)',
                borderWidth: 1
            }
        ]
    };

    async function fetchHeroes() {
        const response = await fetch('https://api.opendota.com/api/heroes');
        heroes = await response.json();
    }

    async function fetchData() {
        const accountId = steamID3;
        const [statsResponse, wlResponse, totalsResponse, matchesResponse, histogramResponse] = await Promise.all([
            fetch(`https://api.opendota.com/api/players/${accountId}`),
            fetch(`https://api.opendota.com/api/players/${accountId}/wl`),
            fetch(`https://api.opendota.com/api/players/${accountId}/totals`),
            fetch(`https://api.opendota.com/api/players/${accountId}/recentMatches`),
            fetch(`https://api.opendota.com/api/players/${accountId}/histograms/kills`)
        ]);
        playerStats = await statsResponse.json();
        playerStats.winLose = await wlResponse.json();
        playerTotals = await totalsResponse.json();
        matchHistory = await matchesResponse.json();
        playerHistogram = await histogramResponse.json();
        updateHistogramChartData();
        paginateMatches();
    }

    function updateHistogramChartData() {
        histogramChartData.labels = playerHistogram.map(item => item.x.toString());
        histogramChartData.datasets[0].data = playerHistogram.map(item => item.games);
    }

    async function fetchMatchDetails(matchId) {
        const response = await fetch(`https://api.opendota.com/api/matches/${matchId}`);
        matchDetails = await response.json();
        viewingMatchDetails = true;
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

    function getHeroName(heroId) {
        const hero = heroes.find(hero => hero.id === heroId);
        return hero ? hero.localized_name : 'Unknown';
    }

    onMount(async () => {
        await fetchHeroes();
    });

    // Chart options
    const options = {
        scales: {
            y: {
                beginAtZero: true
            }
        }
    };

    const data = histogramChartData;
</script>

<style>
    .futuristic-btn {
        background: linear-gradient(45deg, #6dd5ed, #2193b0);
        color: white;
        border: none;
        box-shadow: 0 4px 15px rgba(33, 147, 176, 0.4);
    }
    .futuristic-card {
        background: #1a1a2e;
        color: #e7e7e7;
        border: 1px solid rgba(255, 255, 255, 0.1);
        backdrop-filter: blur(10px);
    }
    .futuristic-header {
        color: #e0e0e0;
        text-shadow: 0 0 10px rgba(33, 147, 176, 0.7);
    }
    :global(body) {
        background-color: #121212;
        color: #ffffff;
    }
    .text-gray-600 {
        color: rgba(255, 255, 255, 0.7);
    }
    .text-green-500 {
        color: #4caf50;
    }
    .text-red-500 {
        color: #f44336;
    }
    .text-blue-600 {
        color: #2196f3;
    }
</style>

<div class="container mx-auto px-4 py-2">
    <div class="mb-6 flex justify-center">
        <Input type="text" bind:value={steamID3} placeholder="Enter SteamID3" class="mr-2 w-full max-w-xs" />
        <Button on:click={fetchData} class="futuristic-btn">Confirm</Button>
    </div>

    {#if !viewingMatchDetails}
        {#if playerStats && playerStats.winLose}
            <div class="futuristic-card shadow-xl rounded-lg p-6 mb-6">
                <h2 class="text-xl font-semibold mb-4 futuristic-header">Player Statistics</h2>
                <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
                    <div class="stat-item">
                        <p class="text-sm font-medium ">MMR Estimate:</p>
                        <p class="text-lg font-bold ml-2">{playerStats.mmr_estimate?.estimate || 'N/A'}</p>
                    </div>
                    <div class="stat-item">
                        <p class="text-sm font-medium ">Wins:</p>
                        <p class="text-lg font-bold text-green-500 ml-2">{playerStats.winLose.win}</p>
                    </div>
                    <div class="stat-item">
                        <p class="text-sm font-medium text-gray-600">Losses:</p>
                        <p class="text-lg font-bold text-red-500 ml-2">{playerStats.winLose.lose}</p>
                    </div>
                    <div class="stat-item">
                        <p class="text-sm font-medium ">Win Rate:</p>
                        <p class="text-lg font-bold text-blue-600 ml-2">
                            {Math.round((playerStats.winLose.win / (playerStats.winLose.win + playerStats.winLose.lose)) * 100) || 'N/A'}%
                        </p>
                    </div>
                    {#each playerTotals as total}
                        <div class="stat-item">
                            <p class="text-sm font-medium ">{total.field}:</p>
                            <p class="text-lg font-bold ml-2">{total.sum}</p>
                        </div>
                    {/each}
                </div>
            </div>
        {/if}

         <!-- Histogram Chart Display -->
    {#if histogramChartData.labels.length}
        <div class="futuristic-card shadow-xl rounded-lg p-6 mb-6 mt-6">
            <h2 class="text-xl font-semibold mb-4 futuristic-header">Player Histogram</h2>
            <Bar {options} {data} />
        </div>
    {/if}

        {#if paginatedMatches.length}
            <div class="mt-4">
                <h2 class="text-lg font-bold mb-4 futuristic-header">Match History</h2>
                <div class="overflow-x-auto relative shadow-md sm:rounded-lg mb-4">
                    <Table hoverable={true}>
                        <TableHead>
                          <TableHeadCell>Match ID</TableHeadCell>
                          <TableHeadCell>Hero</TableHeadCell>
                          <TableHeadCell>K/D/A</TableHeadCell>
                          <TableHeadCell>Result</TableHeadCell>
                        </TableHead>
                        <TableBody class="divide-y">
                          {#each paginatedMatches as match}
                            <TableBodyRow class="bg-white border-b dark:bg-gray-800 dark:border-gray-700 hover:bg-gray-50 dark:hover:bg-gray-600 cursor-pointer" on:click={() => fetchMatchDetails(match.match_id)}>
                              <TableBodyCell>{match.match_id}</TableBodyCell>
                              <TableBodyCell>{getHeroName(match.hero_id)}</TableBodyCell>
                              <TableBodyCell>{match.kills}/{match.deaths}/{match.assists}</TableBodyCell>
                              <TableBodyCell>{match.radiant_win ? 'Win' : 'Loss'}</TableBodyCell>
                            </TableBodyRow>
                          {/each}
                        </TableBody>
                      </Table>
                </div>
                <div class="flex justify-between mt-4">
                    <Button disabled={currentPage <= 1} on:click={() => changePage(currentPage - 1)} class="futuristic-btn">Previous</Button>
                    <Button disabled={(currentPage * rowsPerPage) >= matchHistory.length} on:click={() => changePage(currentPage + 1)} class="futuristic-btn">Next</Button>
                </div>
            </div>
        {/if}
    {:else}
        {#if matchDetails}
        <div class="futuristic-card shadow-xl rounded-lg p-6 mb-6">
            <h2 class="text-2xl font-bold text-gray-800 mb-4 futuristic-header">Match Details</h2>
            <p>Match ID: <span class="font-bold">{matchDetails.match_id}</span></p>
            <p>Duration: <span class="font-bold">{formatDuration(matchDetails.duration)}</span></p>
            <p>Winner: <span class="font-bold">{matchDetails.radiant_win ? 'Radiant' : 'Dire'}</span></p>
            <p>Game Mode: <span class="font-bold">{matchDetails.game_mode}</span></p>
            <p>Lobby Type: <span class="font-bold">{matchDetails.lobby_type}</span></p>
            <h3 class="text-xl font-semibold text-gray-800 mt-6 futuristic-header">Player Performances:</h3>
            <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
                {#each matchDetails.players as player}
                    <Card class="{player.isRadiant ? 'bg-green-500' : 'bg-red-500'} text-white rounded-lg p-4">
                        <p class="font-semibold">{getHeroName(player.hero_id)}</p>
                        <div class="flex justify-between mt-2">
                            <div>
                                <p>Player ID: {player.account_id}</p>
                                <p>Player Name: {player.personaname}</p>
                                <p>K/D/A: {player.kills}/{player.deaths}/{player.assists}</p>
                                <p>GPM: {player.gold_per_min}</p>
                                <p>XPM: {player.xp_per_min}</p>
                                <p>Team: {player.isRadiant ? 'Radiant' : 'Dire'}</p>
                                <p>Wards Placed: {player.sen_placed}</p>
                                <p>Runes: {player.runes}</p>
                                <p>Items: {player.item_0}, {player.item_1}, {player.item_2}, {player.item_3}, {player.item_4}, {player.item_5}</p>
                            </div>
                        </div>
                    </Card>
                {/each}
            </div>
            <Button on:click={viewPlayerStats} class="mt-6 futuristic-btn">Back to Player Stats</Button>
        </div>
    {/if}
    {/if}

</div>
