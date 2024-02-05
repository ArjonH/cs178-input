<script>
    import Calendar from '@event-calendar/core';
    import TimeGrid from '@event-calendar/time-grid';
    import Interaction from '@event-calendar/interaction';
    import '@event-calendar/core/index.css';
    import Modal from '../Modal.svelte';
    import { googleEventsStore } from '../stores.js';
    import { goto } from '$app/navigation';

    let plugins = [TimeGrid, Interaction];
    let googleEvents;
    googleEventsStore.subscribe(value => {
    googleEvents = value;
  });
    let showingGoogle = false;
    let ec;
    let showModal = false;
    let availabilitySure = true;
    let clickedEventId;

    let options = {
        view: 'timeGridWeek',
        events: [],
        selectable: true,
        editable: true,
        select: function (info) {
            addEvent(info)
        },
        eventContent: function(info)  {
            return {html: '<div class="ec-event-time">' + info.timeText + '</div>' + '<div class="ec-event-title">' + info.event.title + '</div>'+ '<button>X</button>'}
        },
        eventClick: function(info) {
            if (info.jsEvent.target === info.el.querySelector('button')) {
                ec.removeEventById(info.event.id);
            }
            else {
                showModal = true;
                clickedEventId = info.event.id;
            }
        },
        eventBackgroundColor: 'green',
        eventTextColor: 'black',
    };

    function addEvent(info) {
        ec.addEvent(info)
        ec = ec.unselect()
    }

    function updateEvent() {
        // Change the event color based on availability level (only if changing to unsure)
        if (! availabilitySure) {
            ec.addEvent(
            {
                ...ec.getEventById(clickedEventId),
                backgroundColor: availabilitySure ? 'green' : 'yellow',
            }
            )
            ec.removeEventById(clickedEventId)
        }
    }

    const CLIENT_ID = '669688591392-rhdn9ebnpq24fc3l08m45ud6tbh8rf4j.apps.googleusercontent.com';
    const API_KEY = 'AIzaSyDDfNxkzJppzzegvfAr9WGc-Y0RzlquirU';

    const DISCOVERY_DOC = 'https://www.googleapis.com/discovery/v1/apis/calendar/v3/rest';
    const SCOPES = 'https://www.googleapis.com/auth/calendar.readonly';
    let tokenClient;
    let gapiInited = false;
    let gisInited = false;

    function gapiLoaded() {
        gapi.load('client', initializeGapiClient);
    }
    async function initializeGapiClient() {
        await gapi.client.init({
          apiKey: API_KEY,
          discoveryDocs: [DISCOVERY_DOC],
        });
        gapiInited = true;
        maybeEnableButtons();
    }
    function gisLoaded() {
        tokenClient = google.accounts.oauth2.initTokenClient({
          client_id: CLIENT_ID,
          scope: SCOPES,
          callback: '', // defined later
        });
        gisInited = true;
        maybeEnableButtons();
    }

    function maybeEnableButtons() {
        if (gapiInited && gisInited) {
          document.getElementById('authorize_button').style.visibility = 'visible';
        }
    }

    function handleSignoutClick() {
        const token = gapi.client.getToken();
        if (token !== null) {
          google.accounts.oauth2.revoke(token.access_token);
          gapi.client.setToken('');
        }
        goto('/');
    }

    function showGoogleEvents() {
        googleEvents.forEach(event => {
            ec.addEvent(
                {
                    title: event.summary,
                    start: event.start.dateTime,
                    end: event.end.dateTime,
                    backgroundColor: 'pink',
                    textColor: 'black',
                    extendedProps: {
                        isGoogle: true
                    }
                }
            )
        });
        showingGoogle = true;
    }

    function hideGoogleEvents() {
        ec.getEvents().forEach(event => { 
            if (event.extendedProps.isGoogle) {
                ec.removeEventById(event.id)
            }
        }); 
        
        showingGoogle = false;
    }
</script>

<svelte:head>
    <script async defer src="https://apis.google.com/js/api.js" on:load={gapiLoaded()}></script>
    <script async defer src="https://accounts.google.com/gsi/client" on:load={gisLoaded()}></script>
</svelte:head>
<button id="signout_button" on:click={() => handleSignoutClick()}>Sign Out</button>

{#if !showingGoogle }
    <button id="show Google calendar" on:click={() => showGoogleEvents()}>Show Google Calendar</button>
{:else}
    <button id="hide Google calendar" on:click={() => hideGoogleEvents()}>Hide Google Calendar</button>
{/if}

<Modal bind:showModal>
    <h2 slot="header">
		Add Information about your Availability
	</h2>
    <span class="space-y-2">
        <label>
            <input
                type="radio"
                name="availability type"
                value={true}
                bind:group={availabilitySure}
            />

            Definitely free
        </label>
        <label>
            <input
                type="radio"
                name="availability type"
                value={false}
                bind:group={availabilitySure}
            />

            Possibly free
        </label>
        <button on:click={() => updateEvent()}>Submit</button>
    </span>
</Modal>

<Calendar bind:this={ec} {plugins} {options} />

