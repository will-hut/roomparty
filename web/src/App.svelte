<script lang="ts">

    import type { ClientMessage } from "../../shared/message";
    import type { ClientGameStateView } from "../../shared/model/game-state-view";
    import { getPlayerId } from "../../shared/model/player";
    import type { CommandError } from "../../shared/command"

    import { ClientWebSocketConnection } from "./client-connection";
    import { countWords } from "./word-cloud";
    import WordCloud from "./WordCloud.svelte";

    const protocol = window.location.protocol === "https:"
        ? "wss:"
        : "ws:";

    const connection = new ClientWebSocketConnection(
        `${protocol}//${window.location.host}/ws`,
    );

    let view: ClientGameStateView | null = null;
    let revision: number = -1;
    let error: CommandError | null = null;

    
    let name = "";
    let freeformText = "";
    let smellRating = 50;

    connection.receive(message => {
        if (message.type === "VIEW") {            
            // Only update the state if the revision is newer than the current state
            if (message.revision > revision) {
                view = message.view;
                error = null;
            }
        }

        if (message.type === "ERROR") {
            error = message.error;
        }
    });

    function send(command: ClientMessage): void {
        connection.send(command);
    }

    function becomePlayer(): void {
        send({
            type: "BECOME",
            role: "PLAYER",
            desiredName: name,
        });
    }

    function becomeHost(): void {
        send({
            type: "BECOME",
            role: "HOST",
            desiredName: name,
        });
    }

    function leave(): void {
        send({
            type: "DESTROY_SELF",
        });
    }

    function submitVote(optionId: string): void { 
        send({
            type: "ADVANCE_INPUT_PAGE",
        });

        send({ 
            type: "CAST_VOTE", 
            vote: 
            { 
                optionId: optionId, 
                freeformText: freeformText,
                rating: smellRating
            }, 
        }); 

        freeformText = "";
        smellRating = 50;
    }
</script>

{#if view}
    <header class="frutiger-card">
        <nav>
            <!-- Logo-->
            <div class="header-left">
                <h1>room party</h1>
            </div>

            {#if view.actor.type === "HOST"}
                <div class="m-1">
                    <button class="frutiger-aero-button" onclick={() => send({ type: "REGRESS_GAME" })}>
                        Regress
                    </button>
                </div>
                <div class="m-1">
                    <button class="pl-2 frutiger-aero-button" onclick={() => send({ type: "ADVANCE_GAME" })}>
                        Advance
                    </button>
                </div>
            {/if}
            

            {#if view.actor.type != "UNASSIGNED"}
                {#if view.actor.type != "HOST" }
                    <div class="header-right pr-2">
                        <p>{view.gamePlayer?.displayName}</p>
                    </div>
                {/if}

                <div class="header-right m-1">
                    <button class="frutiger-aero-button button-red" onclick={leave}>
                        Leave
                    </button>
                </div>
            {/if}
        </nav>
    </header>

    <div class="grid grid-cols-1 lg:grid-cols-4 gap-6 p-6">
        <div class="lg:col-span-3 text-white">
            <!-- Error Banner-->
            {#if error}
                <div class="frutiger-card frutiger-card-rounded frutiger-card-red w-full md:w-1/2 mb-3">
                    <div class="p-3 flex flex-col">
                        <div class="flex flex-row justify-start items-center">
                            <img style="width:3em;" src="images/error-button.svg">
                            <strong class="font-bold">{error.code}</strong>
                        </div>
                        <span class="block sm:inline">{error.message}</span>
                    </div>
                </div>
            {/if}

            <!-- Name + Joining -->
            {#if view.actor.type === "UNASSIGNED"}
                <div class="frutiger-card frutiger-card-rounded w-full p-6">
                    <p class="text-2xl">Join Game</p>
                    <div class="flex max-w-md pt-4">
                        <input bind:value={name} 
                        class="w-full resize-none rounded-lg bg-gray-700 p-3 text-white placeholder-gray-400 outline-none focus:ring-2 focus:ring-yellow-500"
                        placeholder="Name" />
                    </div>
                    <div class="flex max-w-md gap-2 pt-4">
                        <button class="frutiger-aero-button" onclick={becomePlayer}>
                            Become Player
                        </button>

                        {#if view.host === false}
                            <button class="frutiger-aero-button" onclick={becomeHost}>
                                Become Host
                            </button>
                        {/if}
                    </div>
                </div>

            {:else}
                <!-- User View -->
                {#if view.actor.type === "PLAYER" && "currentOptions" in view}
                    <div class="frutiger-card frutiger-card-rounded w-full">
                        <div class="flex flex-col justify-center p-4">
                            <p class="text-2xl"> Scent #{view.currentRound + 1}</p>
                            {#if view.phase === "VOTING"}
                                {#if view.gamePlayer.inputPhase == 0}
                                    <p>Describe the scent and rate it with the slider</p>
                                    <div class="w-full md:w-1/2 pt-5">
                                        <textarea
                                            class="w-full resize-none rounded-lg bg-gray-700 p-3 text-white placeholder-gray-400 outline-none focus:ring-2 focus:ring-yellow-500"
                                            bind:value={freeformText}
                                            placeholder="Type literally anything that comes to mind"
                                            rows="4"
                                        ></textarea>
                                    </div>

                                    <div class = "w-full md:w-1/2 pt-5">
                                        <div class="w-full flex items-center justify-between text-sm text-gray-200">
                                            <span>Smells Horrendous</span>
                                            <span>Smells Amazing</span>
                                        </div>

                                        <input
                                            class="w-full accent-yellow-500 frutiger-aero-range"
                                            type="range"
                                            min="0"
                                            max="100"
                                            bind:value={smellRating}
                                        />
                                    </div>

                                    <button class="frutiger-aero-button ml-auto" onclick={() => send({
                                        type: "ADVANCE_INPUT_PAGE",
                                    })}>Next</button>
                                {:else if view.gamePlayer.inputPhase == 1}
                                    <p>Pick the image that you think most closely represents the scent</p>
                                    <div class="grid lg:grid-cols-4 grid-cols-3 gap-2 p-4">
                                        {#each view.currentOptions as option}
                                            <button
                                                class="rounded font-bold text-white" onclick={() => submitVote(option.id)}
                                            >
                                                <img class = "p-1" src={option.imageLink} alt="" />
                                            </button>
                                        {/each}
                                    </div>
                                {:else if view.gamePlayer.inputPhase == 2}
                                    Meow you submitted now plsz please wait
                                {/if}
                            {:else if view.phase === "RESULTS"}
                                <p1> RESULTS PHASE </p1>
                                <img class="results-image" src="images/butterflies.webp">
                            {:else}
                                <p1> WORDCLOUD </p1>
                            {/if}
                        </div>
                    </div>
                {/if}


                <!-- Host View -->
                {#if view.actor.type === "HOST" && "currentVotes" in view}

                    <h2>Scent {view.currentRound + 1}</h2>

                
                    
                    {#if view.phase === "VOTING"}
                        <div class="mx-auto">
                            <p1> VOTE ON HOW YOU THINK SCENT {view.currentRound + 1} IS </p1>
                        </div>
                        
                        <div class="grid grid-cols-3 gap-2 p-4">
                            <div class="frutiger-card frutiger-card-rounded">
                                <div class="p-3">
                                    <h2> start voting !!! </h2>
                                    <div class="space-y-2">
                                        {#each Object.values(view.players) as player}
                                            {#if player.inputPhase === 0}
                                                <p>{player.displayName}</p>
                                            {/if}
                                        {/each}
                                    </div>
                                </div>
                            </div>

                            <div class="frutiger-card frutiger-card-rounded">
                                <div class="p-3">
                                    <h2> almost done .. </h2>
                                    <div class="space-y-2">
                                        {#each Object.values(view.players) as player}
                                            {#if player.inputPhase === 1}
                                                <p>{player.displayName}</p>
                                            {/if}
                                        {/each}
                                    </div>
                                </div>
                            </div>

                            <div class="frutiger-card frutiger-card-rounded">
                                <div class="p-3">
                                    <h2> done !!! </h2>
                                    <div class="space-y-2">
                                        {#each Object.values(view.players) as player}
                                            {#if player.inputPhase === 2}
                                                <p>{player.displayName}</p>
                                            {/if}
                                        {/each}
                                    </div>
                                </div>
                            </div>
                        </div>
                        

                    {:else if view.phase === "RESULTS"}
                        <div class="frutiger-card frutiger-card-rounded w-full">
                            <p class="text-2xl p-4">Average Rating: {(Object.values(view.currentVotes).reduce((sum, currentVote) => sum + currentVote.rating, 0) / Object.values(view.currentVotes).length).toFixed(2)}</p>

                            <div class="grid grid-cols-6 gap-2 p-4">
                                {#each view.currentOptions as option}
                                    {#if Object.entries(view.currentVotes).some(([playerId, vote]) => vote.optionId == option.id)}
                                        <div class="overflow-hidden rounded-xl border border-gray-700 bg-gray-800">

                                            <!-- Image -->
                                            <img
                                                class="block w-full p-2"
                                                src={option.imageLink}
                                                alt=""
                                            />

                                            <!-- Voters -->
                                            <div class="px-3 py-2">
                                                {#each Object.entries(view.currentVotes) as [playerId, vote]}
                                                    {#if vote.optionId === option.id}
                                                        <div class="text-sm font-medium text-white">
                                                            {view.players[playerId]?.displayName}
                                                        </div>
                                                    {/if}
                                                {/each}
                                            </div>

                                        </div>
                                    {/if}
                                {/each}
                            </div>
                        </div>
                    {:else}
                        <div class="frutiger-card frutiger-card-rounded w-full">
                            <WordCloud
                            words={countWords(
                                Object.values(view.currentVotes)
                                    .map(vote => vote.freeformText)
                                    .filter(text => text.trim().length > 0)
                                )}
                            />
                        </div>
                    {/if}

                {/if}

            {/if}
        </div>

        <!-- Players List -->
        <div class="flex flex-col items-center justify-left">
            <div class="frutiger-card frutiger-card-rounded w-full">
                <div class="p-3">
                    <h2>players</h2>

                    <div class="space-y-2">
                        {#each Object.values(view.players) as player}
                            <div class="flex items-center player-list-item">
                                <p class="flex-1 font-medium text-white flex flex-row justify-left items-center">
                                    <img class="buddy-img" src="images/Buddy-bleu.png">
                                    {player.displayName}
                                </p>

                                {#if view.actor.type === "HOST"}
                                    <button
                                        class="frutiger-aero-button button-red"
                                        onclick={() => send({
                                            type: "DESTROY_PLAYER",
                                            playerId: getPlayerId(player.displayName)
                                        })}
                                    >
                                        Kick
                                    </button>
                                {/if}
                            </div>
                        {:else}
                            <p class="text-white">No players</p>
                        {/each}
                    </div>
                </div>
            </div>
        </div>
    </div>
{/if}
