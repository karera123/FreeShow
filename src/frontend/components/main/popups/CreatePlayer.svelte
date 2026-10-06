<script lang="ts">
    import { tick } from "svelte"
    import { uid } from "uid"
    import { activePopup, playerVideos, popupData } from "../../../stores"
    import { newToast } from "../../../utils/common"
    import { translateText } from "../../../utils/language"
    import { registerPopupSubmit } from "../../../utils/popup"
    import { getUrlTimestamp, getVimeoData, getYouTubeData, trimPlayerId } from "../../drawer/player/playerHelper"
    import { clone } from "../../helpers/array"
    import { addVideoMarker } from "../../helpers/media"
    import Icon from "../../helpers/Icon.svelte"
    import T from "../../helpers/T.svelte"
    import { joinTime, secondsToTime } from "../../helpers/time"
    import MaterialButton from "../../inputs/MaterialButton.svelte"
    import MaterialTextInput from "../../inputs/MaterialTextInput.svelte"
    import Loader from "../Loader.svelte"

    const ID_INPUT = "playerVideoId"
    const NAME_INPUT = "playerVideoName"

    let active: "youtube" | "vimeo" = $popupData.active
    let editId: string = $popupData.id || ""
    $: if (active) popupData.set({})

    registerPopupSubmit(submit)

    const currentId = editId || uid()
    // local draft, only stored when added
    let data = clone($playerVideos[editId] || { name: "", id: "" })

    let pendingMarkerTime = 0
    let autoName = ""
    let loadingCount = 0

    // a name is required
    $: canAdd = !!data.id && !!data.name?.trim() && !loadingCount

    function confirmId(value: string) {
        const id = trimPlayerId(value || "", active)
        if (!id || id === data.id) return

        // extract any timestamp in the pasted link (e.g. ?t=1234)
        pendingMarkerTime = getUrlTimestamp(value)

        data = { ...data, id }
        loadName(id)
    }

    // manually get the default name again
    function refreshName() {
        if (!data.id) return
        loadName(data.id, true)
    }

    async function loadName(id: string, replace = false) {
        let newName = ""
        loadingCount++
        try {
            if (active === "youtube") newName = (await getYouTubeData(id)).name
            else if (active === "vimeo") newName = (await getVimeoData(id)).name
        } finally {
            loadingCount--
        }

        // another id was entered while loading
        if (data.id !== id) return

        // don't replace a manually entered name
        if (newName && (replace || !data.name || data.name === autoName)) {
            autoName = newName
            data = { ...data, name: newName }
        }

        await tick()
        const nameInput = document.getElementById(NAME_INPUT) as HTMLInputElement | null
        nameInput?.focus()
        nameInput?.select()
    }

    function submit() {
        if (editId) return activePopup.set(null)

        // enter in the id field confirms the link
        const focused = document.activeElement as HTMLInputElement | null
        if (focused?.id === ID_INPUT) {
            const previousId = data.id
            confirmId(focused.value)
            if (data.id !== previousId) return
        }

        if (!canAdd) return

        playerVideos.update((a) => {
            a[currentId] = { ...data, name: data.name.trim(), type: active }
            return a
        })

        if (pendingMarkerTime) {
            const markerIndex = addVideoMarker(currentId, pendingMarkerTime, "URL")
            if (markerIndex > -1) newToast(translateText("actions.time_marker_added", null, [joinTime(secondsToTime(pendingMarkerTime))]))
        }

        activePopup.set(null)
    }
</script>

<MaterialTextInput id={ID_INPUT} label="inputs.video_id" value={data.id || ""} placeholder="e.g. X-AJdKty74M" disabled={!!(data.id && editId)} on:change={(e) => confirmId(e.detail)} autofocus={!data.id} pasteBtn={!data.id} />
{#if !editId}
    <MaterialTextInput id={NAME_INPUT} label="inputs.name" value={data.name} disabled={!data.id} on:input={(e) => (data.name = e.detail)}>
        <svelte:fragment slot="buttons">
            {#if loadingCount}
                <div class="loading"><Loader size={0.5} /></div>
            {:else}
                <MaterialButton title="meta.autofill" disabled={!data.id} on:click={refreshName} white>
                    <Icon id="refresh" white />
                </MaterialButton>
            {/if}
        </svelte:fragment>
    </MaterialTextInput>

    <MaterialButton variant="contained" style="margin-top: 20px;" icon="add" disabled={!canAdd} on:click={submit}>
        <T id="settings.add" />
    </MaterialButton>
{/if}

<style>
    .loading {
        display: flex;
        padding: 0.75rem;
    }
</style>
