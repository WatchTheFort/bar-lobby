<template>
    <div class="playerlist" :class="{ dragging: draggedParticipant !== null }">
        <div
            v-for="[teamId, contenders] in battle.teams.value"
            :key="`team${teamId}`"
            class="group"
            data-type="group"
            @dragenter.prevent="dragEnter($event, teamId)"
            @dragover.prevent
            @dragleave.prevent="dragLeave($event, teamId)"
            @drop="onDrop($event, teamId)"
        >
            <div class="flex-row gap-md">
                <div class="title">Team {{ teamId + 1 }} ({{ contenders.length }})</div>
                <Button class="slim" :flexGrow="false" @click="addBot(teamId)"> Add bot </Button>
                <Button v-if="me.battleStatus.isSpectator || me.battleStatus.teamId !== teamId" class="slim" @click="joinTeam(teamId)"> Join </Button>
            </div>
            <div class="participants">
                <div v-for="(contender, contenderIndex) in contenders" :key="`contender${contenderIndex}`" draggable @dragstart="dragStart($event, contender)" @dragend="dragEnd($event, contender)">
                    <PlayerParticipant v-if="'userId' in contender" :battle="battle" :player="contender" />
                    <BotParticipant v-else :battle="battle" :bot="contender" />
                </div>
            </div>
        </div>
        <div
            class="group"
            data-type="group"
            @dragenter.prevent="dragEnter($event, emptyTeamId)"
            @dragover.prevent
            @dragleave.prevent="dragLeave($event, emptyTeamId)"
            @drop="onDrop($event, emptyTeamId)"
        >
            <div class="flex-row gap-md">
                <div class="title">Team {{ emptyTeamId + 1 }}</div>
                <Button class="slim" @click="addBot(emptyTeamId)"> Add bot </Button>
                <Button class="slim" @click="joinTeam(emptyTeamId)"> Join </Button>
            </div>
        </div>
        <div class="group" data-type="group" @dragenter.prevent="dragEnter($event)" @dragover.prevent @dragleave.prevent="dragLeave($event)" @drop="onDrop($event)">
            <div class="flex-row gap-md">
                <div class="title">Spectators ({{ battle.spectators.value.length }})</div>
                <Button v-if="!me.battleStatus.isSpectator" class="slim" @click="joinTeam()"> Join </Button>
            </div>
            <div class="participants">
                <div
                    v-for="(spectator, spectatorIndex) in battle.spectators.value"
                    :key="`spectator${spectatorIndex}`"
                    draggable
                    @dragstart="dragStart($event, spectator)"
                    @dragend="dragEnd($event, spectator)"
                >
                    <PlayerParticipant :battle="battle" :player="spectator" />
                </div>
            </div>
        </div>
    </div>
</template>

<script lang="ts" setup>
import { computed } from "@vue/reactivity";
import { randomFromArray } from "jaz-ts-utils";
import { Ref, ref } from "vue";

import BotParticipant from "@/components/battle/BotParticipant.vue";
import PlayerParticipant from "@/components/battle/PlayerParticipant.vue";
import Button from "@/components/controls/Button.vue";
import { aiNames } from "@/config/ai-names";
import { AbstractBattle } from "@/model/battle/abstract-battle";
import { Bot, Faction } from "@/model/battle/types";
import { CurrentUser, User } from "@/model/user";

const props = defineProps<{
    battle: AbstractBattle;
    me: CurrentUser;
}>();

const emptyTeamId = computed(() => {
    const teams = props.battle.teams.value;
    for (let i = 0; i <= teams.size; i++) {
        if (!teams.get(i)) {
            return i;
        }
    }
    return -1;
});

const addBot = (teamId: number) => {
    let randomName = randomFromArray(aiNames);
    while (props.battle.bots.some((bot) => bot.name === randomName)) {
        randomName = randomFromArray(aiNames);
    }

    props.battle.addBot({
        playerId: props.battle.contenders.value.length,
        teamId,
        name: randomName!,
        aiShortName: "BARb",
        faction: Faction.Armada,
        ownerUserId: props.me.userId,
        aiOptions: {},
    });
};

const joinTeam = (teamId?: number) => {
    if (props.me.battleStatus.isSpectator && teamId !== undefined) {
        props.battle.spectatorToPlayer(props.me, teamId);
    } else if (!props.me.battleStatus.isSpectator && teamId === undefined) {
        props.battle.playerToSpectator(props.me);
    } else if (!props.me.battleStatus.isSpectator && teamId !== undefined) {
        props.battle.setContenderTeam(props.me, teamId);
    }
};

let draggedParticipant: Ref<User | Bot | null> = ref(null);
let draggedEl: Element | null = null;

const dragEnter = (event: DragEvent, teamId?: number) => {
    if (!draggedParticipant.value) {
        return;
    }

    const target = event.target as HTMLElement;
    const groupEl = target.closest("[data-type=group]");
    if (draggedEl && groupEl) {
        document.querySelectorAll("[data-type=group]").forEach((el) => {
            el.classList.remove("highlight");
        });
    }

    const draggingContenderToOwnTeam =
        ("userId" in draggedParticipant.value && !draggedParticipant.value.battleStatus.isSpectator && draggedParticipant.value.battleStatus.teamId === teamId) ||
        (!("userId" in draggedParticipant.value) && draggedParticipant.value.teamId === teamId);
    const draggingSpectatorToSpectator = "userId" in draggedParticipant.value && draggedParticipant.value.battleStatus.isSpectator && teamId === undefined;
    const draggingBotToSpectator = !("userId" in draggedParticipant.value) && teamId === undefined;
    if (draggingContenderToOwnTeam || draggingSpectatorToSpectator || draggingBotToSpectator) {
        // TODO: disable drag cursor
        return;
    }

    if (groupEl) {
        groupEl.classList.add("highlight");
    }
};

const dragLeave = (event: DragEvent, teamId?: number) => {
    if (!draggedParticipant.value) {
        return;
    }

    const draggingContenderToOwnTeam =
        ("userId" in draggedParticipant.value && !draggedParticipant.value.battleStatus.isSpectator && draggedParticipant.value.battleStatus.teamId === teamId) ||
        (!("userId" in draggedParticipant.value) && draggedParticipant.value.teamId === teamId);
    const draggingSpectatorToSpectator = "userId" in draggedParticipant.value && draggedParticipant.value.battleStatus.isSpectator && teamId === undefined;
    const draggingBotToSpectator = !("userId" in draggedParticipant.value) && teamId === undefined;
    if (draggingContenderToOwnTeam || draggingSpectatorToSpectator || draggingBotToSpectator) {
        // TODO: disable drag cursor
        return;
    }
};

const dragStart = (event: DragEvent, participant: User | Bot) => {
    draggedParticipant.value = participant;
    draggedEl = event.target as Element;
    const participantEl = draggedEl?.querySelector("[data-type=participant]");
    if (participantEl) {
        participantEl.classList.add("dragging");
    }
};

const dragEnd = (event: DragEvent, participant: User | Bot) => {
    const participantEl = draggedEl?.querySelector("[data-type=participant]");
    if (participantEl) {
        participantEl.classList.remove("dragging");
    }
    draggedParticipant.value = null;
    draggedEl = null;

    document.querySelectorAll("[data-type=group]").forEach((el) => {
        el.classList.remove("highlight");
    });
};

const onDrop = (event: DragEvent, teamId?: number) => {
    const target = event.target as Element;
    if (target.getAttribute("data-type") === "group" && draggedParticipant.value) {
        if (teamId !== undefined && (("userId" in draggedParticipant.value && !draggedParticipant.value.battleStatus.isSpectator) || !("userId" in draggedParticipant.value))) {
            props.battle.setContenderTeam(draggedParticipant.value, teamId);
        } else if ("userId" in draggedParticipant.value && !draggedParticipant.value.battleStatus.isSpectator) {
            props.battle.playerToSpectator(draggedParticipant.value);
        } else if (teamId !== undefined && "userId" in draggedParticipant.value && draggedParticipant.value.battleStatus.isSpectator) {
            props.battle.spectatorToPlayer(draggedParticipant.value, teamId);
        }
    }
};
</script>

<style lang="scss" scoped>
.playerlist {
    display: flex;
    flex-direction: column;
    overflow-y: auto;
    padding-right: 5px;
    &.dragging .group > * {
        pointer-events: none;
    }
}
.group {
    position: relative;
    &:not(:last-child):after {
        content: "";
        display: flex;
        background: rgba(255, 255, 255, 0.1);
        width: 100%;
        height: 1px;
        margin: 10px 0;
    }
    &.highlight {
        &:before {
            @extend .fullsize;
            width: calc(100% + 10px);
            height: calc(100%);
            left: -5px;
            top: -5px;
            background: rgba(255, 255, 255, 0.1);
        }
    }
}
.title {
    font-size: 26px;
}
.participants {
    display: flex;
    flex-direction: row;
    gap: 5px;
    flex-wrap: wrap;
    margin-top: 5px;
}
</style>
