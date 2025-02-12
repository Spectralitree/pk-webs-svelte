<script lang="ts">
    import { Container, Row, Col, Alert, Spinner, Card, CardHeader, CardBody, Accordion, AccordionItem, CardTitle } from "sveltestrap";
    import Body from '../lib/group/Body.svelte';
    import MemberBody from '../lib/member/Body.svelte';
    import { useParams, Link } from 'svelte-navigator';
    import { onMount } from 'svelte';
    import api from "../api";
    import { Member, Group } from "../api/types";
    import CardsHeader from "../lib/CardsHeader.svelte";
    import FaUsers from 'svelte-icons/fa/FaUsers.svelte';
    import FaList from 'svelte-icons/fa/FaList.svelte';
    import FaUserCircle from 'svelte-icons/fa/FaUserCircle.svelte';
    import ListPagination from '../lib/ListPagination.svelte';
    import FaLock from 'svelte-icons/fa/FaLock.svelte'

    let loading = true;
    let memberLoading = false;
    let params = useParams();
    let err = "";
    let memberErr = "";
    let group: Group;
    let members: Member[] = [];
    let systemMembers: Group[] = [];
    let isMainDash = false;
    let isDeleted = false;
    let notOwnSystem = false;

    const isPage = true;
    export let isPublic = true;
    let settings = JSON.parse(localStorage.getItem("pk-settings"));

    let currentPage = 1;
    let itemsPerPage = settings && settings.accessibility && settings.accessibility.expandedcards ? 5 : 10;

    $: indexOfLastItem = currentPage * itemsPerPage;
    $: indexOfFirstItem = indexOfLastItem - itemsPerPage;
    $: pageAmount = Math.ceil(members.length / itemsPerPage);

    $: orderedMembers = members.sort((a, b) => a.name.localeCompare(b.name));
    $: slicedMembers = orderedMembers.slice(indexOfFirstItem, indexOfLastItem);

    onMount(() => {
        fetchGroup();
    });

    async function fetchGroup() {
        try {
            group = await api().groups($params.id).get({auth: !isPublic});
            if (!isPublic && !group.privacy) {
                notOwnSystem = true;
                throw new Error("Group is not from own system.");
            }
            err = "";
            loading = false;
            memberLoading = true;
            await new Promise(resolve => setTimeout(resolve, 1000));
            fetchMembers();
        } catch (error) {
            console.log(error);
            err = error.message;
            loading = false;
        }
    }

    async function fetchMembers() {
        try {
            members = await api().groups($params.id).members().get({auth: !isPublic});
            group.members = members.map(function(member) {return member.uuid});
            if (!isPublic) {
                await new Promise(resolve => setTimeout(resolve, 1000));
                systemMembers = await api().systems("@me").members.get({ auth: true });
            }
            memberErr = "";
            memberLoading = false;
        } catch (error) {
            console.log(error);
            memberErr = error.message;
            memberLoading = false;
        }
    }

    async function updateMembers() {
      memberLoading = true;
      await new Promise(resolve => setTimeout(resolve, 500));
      fetchMembers();
    }

    function updateDelete() {
        isDeleted = true;
    }
    
    function updateMemberList(event: any) {
        members = members.map(member => member.id !== event.detail.id ? member : event.detail);
        systemMembers = systemMembers.map(member => member.id !== event.detail.id ? member : event.detail);
    }

    function deleteMemberFromList(event: any) {
        members = members.filter(member => member.id !== event.detail);
        systemMembers = systemMembers.filter(member => member.id !== event.detail);
  }
</script>

{#if settings && settings.appearance.color_background && !notOwnSystem}
    <div class="background" style="background-color: {group && `#${group.color}`}"></div>
{/if}
{#if group && group.banner && settings && settings.appearance.banner_top && !notOwnSystem}
<div class="banner" style="background-image: url({group.banner})" />
{/if}
<Container>
    <Row>
        <Col class="mx-auto" xs={12} lg={11} xl={10}>
            {#if isDeleted}
                <Alert color="success">Group has been successfully deleted. <Link to="/dash">Return to dash</Link></Alert>
            {:else}
            {#if isPublic}
                <Alert color="info">You are currently <b>viewing</b> a group.</Alert>
            {/if}
            {#if notOwnSystem}
                <Alert color="danger">This group does not belong to your system, did you mean to look up <Link to={`/profile/g/${group.id}`}>it's public page</Link>?</Alert>
            {:else if err}
                <Alert color="danger">{@html err}</Alert>
            {:else if loading}
                <Spinner/>
            {:else if group && group.id}
                <Card class="mb-4">
                    <CardHeader>
                        <CardsHeader bind:item={group}>
                            <FaUsers slot="icon" />
                        </CardsHeader>
                    </CardHeader>
                    <CardBody>
                        <Body on:deletion={updateDelete} on:updateMembers={updateMembers} bind:members={systemMembers} bind:group={group} isPage={isPage} isPublic={isPublic}/>
                    </CardBody>
                </Card>
            {/if}
            {#if memberLoading}
                <Alert color="primary"><Spinner size="sm" /> Fetching members...</Alert>
            {:else if memberErr}
                <Alert color="danger">{memberErr}</Alert>
            {:else if members && members.length > 0}
            <Card class="mb-2">
                <CardHeader>
                    <CardTitle style="margin-top: 8px; outline: none;">
                        <div class="icon d-inline-block">
                            <FaList />
                        </div> Group list
                    </CardTitle>
                </CardHeader>
            </Card>
            <ListPagination bind:currentPage bind:pageAmount />
            {#if settings && settings.accessibility ? (!settings.accessibility.expandedcards && !settings.accessibility.pagelinks) : true}
            <Accordion class="mb-3" stayOpen>
            {#each slicedMembers as member, index (member.id)}
            {#if (!isPublic && member.privacy.visibility === "public") || isPublic}
                <AccordionItem>
                    <CardsHeader bind:item={member} slot="header">
                        <FaUserCircle slot="icon" />
                    </CardsHeader>
                    <MemberBody on:update={updateMemberList} isMainDash={isMainDash} on:deletion={deleteMemberFromList} bind:member bind:isPublic={isPublic}/>
                </AccordionItem>
                {:else}
                <AccordionItem>
                    <CardsHeader bind:item={member} slot="header">
                        <FaLock slot="icon" />
                    </CardsHeader>
                    <MemberBody on:update={updateMemberList} isMainDash={isMainDash} on:deletion={deleteMemberFromList} bind:member bind:isPublic={isPublic}/>
                </AccordionItem>
                {/if}
            {/each}
            </Accordion>
            {:else if settings.accessibility.expandedcards}
            {#each slicedMembers as member, index (member.id)}
                {#if (!isPublic && member.privacy.visibility === "public") || isPublic}
                <Card class="mb-3">
                    <CardHeader>
                        <CardsHeader bind:item={member}>
                            <FaUserCircle slot="icon" />
                        </CardsHeader>
                    </CardHeader>
                    <CardBody>
                        <MemberBody on:update={updateMemberList} isMainDash={isMainDash} on:deletion={deleteMemberFromList} bind:member bind:isPublic={isPublic}/>
                    </CardBody>
                </Card>
                {:else}
                <Card class="mb-3">
                    <CardHeader>
                        <CardsHeader bind:item={member}>
                            <FaLock slot="icon" />
                        </CardsHeader>
                    </CardHeader>
                    <CardBody>
                        <MemberBody on:update={updateMemberList} isMainDash={isMainDash} on:deletion={deleteMemberFromList} bind:member bind:isPublic={isPublic}/>
                    </CardBody>
                </Card>
                {/if}
            {/each}
            {:else}
            <div class="my-3">
            {#each slicedMembers as member, index (member.id)}
                {#if (!isPublic && member.privacy.visibility === "public") || isPublic}
                <Card>
                    <Link class="accordion-button collapsed" style="text-decoration: none;" to={!isPublic ? `/dash/m/${member.id}` : `/profile/m/${member.id}`}>
                        <CardsHeader bind:item={member}>
                            <FaUserCircle slot="icon" />
                        </CardsHeader>
                    </Link>
                </Card>
                {:else}
                <Card>
                    <Link class="accordion-button collapsed" style="text-decoration: none;" to={!isPublic ? `/dash/m/${member.id}` : `/profile/m/${member.id}`}>
                        <CardsHeader bind:item={member}>
                            <FaLock slot="icon" />
                        </CardsHeader>
                    </Link>
                </Card>
                {/if}
            {/each}
            </div>
            {/if}
            <ListPagination bind:currentPage bind:pageAmount />
            {/if}
            {/if}
        </Col>
    </Row>
</Container>

<style>
    .background {
        position: fixed;
        top: 0;
        left: 0;
        width: 100%;
        flex: 1;
        min-height: 100%;
        z-index: -30;
    }
</style>

