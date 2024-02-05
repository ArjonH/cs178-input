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

