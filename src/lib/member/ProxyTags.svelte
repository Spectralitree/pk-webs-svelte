<script lang="ts">
    import { createEventDispatcher } from "svelte";
    import { Col, Row, Input, Label, Button, Alert, Spinner, InputGroup } from "sveltestrap";

    import { Member } from '../../api/types';
    import api from '../../api';

    let loading: boolean;
    export let proxyOpen: boolean;
    export let member: Member;
    const toggleProxyModal = () => (proxyOpen = !proxyOpen);

    let err: string;

    const dispatch = createEventDispatcher();

    function update() {
        dispatch('update', member);
    }

    let input = member.proxy_tags;

    async function submit() {
        err = null;
        if (input.some(tag => tag.prefix && tag.suffix && tag.prefix.length + tag.suffix.length + 4 > 100)) {
            err = "One of your proxy tags is too long (prefix + 'text' + suffix must be shorter than 100 characters). Please shorten this tag, or remove it."
            return;
        }

        let data: Member = {proxy_tags: input};
        loading = true;

        try {
            let res = await api().members(member.id).patch({data});
            member = res;
            err = null;
            update();
            loading = false;
            toggleProxyModal();
        } catch (error) {
            console.log(error);
            err = error.message;
            loading = false;
        }
    }
</script>

{#if err}
<Alert color="danger">{err}</Alert>
{/if}
<Row class="mb-2">
    {#each input as proxyTag, index (index)}
    <Col xs={12} lg={6} class="mb-2">
        <InputGroup>
            <Input style="resize: none; height: 1em" type="textarea" bind:value={proxyTag.prefix} />
            <Input disabled value="text"/>
            <Input style="resize: none; height: 1em" type="textarea" bind:value={proxyTag.suffix}/>
        </InputGroup>
    </Col>
    {/each}
    <Col xs={12} lg={6} class="mb-2">
        <Button class="w-100" color="secondary" on:click={() => {input.push({prefix: "", suffix: ""}); input = input;}}>New</Button>
    </Col>
</Row>
{#if !loading}<Button style="flex 0" color="primary" on:click={submit}>Submit</Button> <Button style="flex: 0" color="secondary" on:click={toggleProxyModal}>Back</Button>
{:else}<Button style="flex 0" color="primary" disabled><Spinner size="sm"/></Button> <Button style="flex: 0" color="secondary" disabled>Back</Button>
{/if}