<script>
    import Calendar from '@event-calendar/core';
    import TimeGrid from '@event-calendar/time-grid';
    import Interaction from '@event-calendar/interaction';
    import '@event-calendar/core/index.css';
    import { nameStore } from '../stores.js';
    import { Button, Container, Modal, ModalBody, ModalFooter, Row, Col, ListGroup, ListGroupItem } from '@sveltestrap/sveltestrap';

    let plugins = [TimeGrid, Interaction];
    let googleEvents;
    let name;
    nameStore.subscribe(value => {
        name = value;
    });
    let showingGoogle = false;
    let ec;
    let showEditEventModal = false;
    $: availabilitySure = true;
    let clickedEventId;
    let googleLogin = false;

    const CLIENT_ID = '669688591392-rhdn9ebnpq24fc3l08m45ud6tbh8rf4j.apps.googleusercontent.com';
    const API_KEY = 'AIzaSyDDfNxkzJppzzegvfAr9WGc-Y0RzlquirU';

    const DISCOVERY_DOC = 'https://www.googleapis.com/discovery/v1/apis/calendar/v3/rest';
    const SCOPES = 'https://www.googleapis.com/auth/calendar.readonly';
    let tokenClient;
    let gapiInited = false;
    let gisInited = false;

    let options = {
        view: 'timeGridWeek',
        events: [],
        selectable: true,
        editable: true,
        select: function (info) {
            addEvent(info)
        },
        eventContent: function(info)  {
            let buttonHtml = info.event.extendedProps.isGoogle ? '' : '<Button>X</Button>';
            return {html: '<div class="ec-event-time">' + info.timeText + '</div>' + '<div class="ec-event-title">' + info.event.title + '</div>'+ buttonHtml}
        },
        eventClick: function(info) {
            if (info.jsEvent.target === info.el.querySelector('button')) {
                ec.removeEventById(info.event.id);
            }
            else {
                showEditEventModal = true;
                clickedEventId = info.event.id;
            }
        },
        eventBackgroundColor: '#158535',
        eventTextColor: 'black',
    };

    function addEvent(info) {
        ec.addEvent(info)
        ec = ec.unselect()
    }

    function updateEvent() {
        // Change the event color based on availability level
        ec.addEvent(
            {
                ...ec.getEventById(clickedEventId),
                backgroundColor: availabilitySure ? '#158535' : '#dde83c',
            }
        )
        ec.removeEventById(clickedEventId)
        showEditEventModal = false;
    }

    function handleSignoutClick() {
        const token = gapi.client.getToken();
        if (token !== null) {
          google.accounts.oauth2.revoke(token.access_token);
          gapi.client.setToken('');
        }
        hideGoogleEvents();
        googleLogin=false;
    }

    function showGoogleEvents() {
        googleEvents.forEach(event => {
            ec.addEvent(
                {
                    title: event.summary,
                    start: event.start.dateTime,
                    end: event.end.dateTime,
                    backgroundColor: 'pink',
                    display: 'background',
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

    function gapiLoaded() {
        gapi.load('client', initializeGapiClient);
    }

    async function initializeGapiClient() {
        await gapi.client.init({
          apiKey: API_KEY,
          discoveryDocs: [DISCOVERY_DOC],
        });
        gapiInited = true;
    }

    function gisLoaded() {
        tokenClient = google.accounts.oauth2.initTokenClient({
          client_id: CLIENT_ID,
          scope: SCOPES,
          callback: '', // defined later
        });
        gisInited = true;
    }

    function handleAuthClick() {
        tokenClient.callback = async (resp) => {
          if (resp.error !== undefined) {
            throw (resp);
          }
          googleLogin=true;
          await listUpcomingEvents();
        };
        if (gapi.client.getToken() === null) {
          // Prompt the user to select a Google Account and ask for consent to share their data
          // when establishing a new session.
          tokenClient.requestAccessToken({prompt: 'consent'});
        } else {
          // Skip display of account chooser and consent dialog for an existing session.
          tokenClient.requestAccessToken({prompt: ''});
        }
    }

    async function listUpcomingEvents() {
        let response;
        try {
          const request = {
            'calendarId': 'primary',
            'timeMin': (new Date()).toISOString(),
            'showDeleted': false,
            'singleEvents': true,
            'maxResults': 15,
            'orderBy': 'startTime',
          };
          response = await gapi.client.calendar.events.list(request);
        } catch (err) {
          document.getElementById('content').innerText = err.message;
          return;
        }
        const events = response.result.items;
        if (!events || events.length == 0) {
          return;
        }
        googleEvents = events;
    }

</script>

<svelte:head>
    <script async defer src="https://apis.google.com/js/api.js" on:load={gapiLoaded()}></script>
    <script async defer src="https://accounts.google.com/gsi/client" on:load={gisLoaded()}></script>
</svelte:head>

<Container>
    <Row>
        <Col xs="auto">
            <h1>{name}'s availability</h1>
        </Col>
    </Row>
    <Row>
        <h2>How to use:</h2>
        <ListGroup numbered>
            <ListGroupItem>To show your availability, click and drag directly on the calendar. You can move an availability window or extend the window.</ListGroupItem>
            <ListGroupItem>To remove an event click on the X icon directly on the event</ListGroupItem>
            <ListGroupItem>To change the availability status of an event (from definitely available to possibly free) click directly on the event</ListGroupItem>
            <ListGroupItem>If you logged in with your Google Account, you can display your Google calendar events to schedule around.</ListGroupItem>
            <ListGroupItem>To log in with your Google Account, click Authorize Google Account</ListGroupItem>
            <ListGroupItem>Once you are logged in you will have the option to display your Google Calendar events. These are just a placeholder and will not be sent to other users. 
                Having a Google Event at a time also does not prevent you from choosing that as an available time.</ListGroupItem>
            <ListGroupItem>If your Google account events are being displayed, you can also toggle this off to see the calendar without them</ListGroupItem>
        </ListGroup>
    </Row>
    <Row>
        <Col>
            {#if !googleLogin }
                <Button color="success" id="authorize_button" on:click={() => handleAuthClick()}>Authorize Google Account</Button>
            {/if}
            {#if !showingGoogle && googleLogin }
                <Button color="success" id="show Google calendar" on:click={() => showGoogleEvents()}>Show Google Calendar</Button>
            {:else if googleLogin}
                <Button color="warning" id="hide Google calendar" on:click={() => hideGoogleEvents()}>Hide Google Calendar</Button>
            {/if}
            {#if googleLogin }
                <Button color="danger" id="signout_button" on:click={() => handleSignoutClick()}>Sign Out of Google Account</Button>
            {/if}
        </Col>
    </Row>
</Container>

<Modal body header="Add Information about your Availability" isOpen={showEditEventModal} {updateEvent}>
    <ModalBody>
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
        <Button color="success" on:click={() => updateEvent()}>Submit</Button>
    </ModalBody>
    <ModalFooter>
        <Button color="info" on:click={() => showEditEventModal=false}>Close</Button>
    </ModalFooter>
</Modal>

<Calendar bind:this={ec} {plugins} {options} /> 
