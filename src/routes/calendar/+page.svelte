<script>
    import Calendar from '@event-calendar/core';
    import TimeGrid from '@event-calendar/time-grid';
    import Interaction from '@event-calendar/interaction';
    import '@event-calendar/core/index.css';
    import { nameStore } from '../stores.js';
    import { Button, Container, Modal, ModalBody, ModalFooter, Row, Col, ListGroup, ListGroupItem, Icon, ModalHeader } from '@sveltestrap/sveltestrap';

    let plugins = [TimeGrid, Interaction];
    let googleEvents; // store for the events imported from Google calendar
    // Get the name the user inputted in previous page
    let name;
    nameStore.subscribe(value => {
        name = value;
    });
    let showingGoogle = false; // Variable keeping track of if Google calendar is showing
    let ec;
    let showEditEventModal = false; // Variable that toggles to show or close modal
    let availabilitySure = true; // Variable to determine degree of availability
    $: availabilitySure
    let clickedEventId; // The ID of the last clicked event
    let googleLogin = false; // Stores if user is logged into their Google account

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
        slotMinTime: '9:00:00', //Have calendar start at 9am
		slotMaxTime: '23:00:00', //End at 11pm
        // When a user makes a selection, add that event
        select: function (info) {
            addEvent(info)
        },
        // Add a X button on each event so user can delete event
        eventContent: function(info)  {
            let buttonHtml = info.event.extendedProps.isGoogle ? '' : '<Button>X</Button>';
            return {html: '<div class="ec-event-time">' + info.timeText + '</div>' + '<div class="ec-event-title">' + info.event.title + '</div>'+ buttonHtml}
        },
        eventClick: function(info) {
            // If user clicks the X button delete the event
            if (info.jsEvent.target === info.el.querySelector('button')) {
                ec.removeEventById(info.event.id);
            }
            // If user clicks the event show the edit event modal
            else {
                showEditEventModal = true;
                clickedEventId = info.event.id;
            }
        },
        eventBackgroundColor: '#158535',
        eventTextColor: 'black',
    };

    // When a user makes a selection on the calendar add an event and unselect
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
        availabilitySure = true;
    }

    // Sign out of Google account
    function handleSignoutClick() {
        const token = gapi.client.getToken();
        if (token !== null) {
          google.accounts.oauth2.revoke(token.access_token);
          gapi.client.setToken('');
        }
        hideGoogleEvents();
        googleLogin=false;
    }

    // Display the user's Google events on the calendar
    function showGoogleEvents() {
        googleEvents.forEach(event => {
            ec.addEvent(
                {
                    title: event.summary,
                    start: event.start.dateTime,
                    end: event.end.dateTime,
                    backgroundColor: 'red',
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

    // Hide the user's Google events from the calendar
    function hideGoogleEvents() {
        ec.getEvents().forEach(event => { 
            if (event.extendedProps.isGoogle) {
                ec.removeEventById(event.id)
            }
        }); 
        
        showingGoogle = false;
    }

    //Functions to authorize Google account:
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

    // Log in to user's Google account
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

    // Get user's Google events
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
        showGoogleEvents();
    }

</script>

<svelte:head>
    <script async defer src="https://apis.google.com/js/api.js" on:load={gapiLoaded()}></script>
    <script async defer src="https://accounts.google.com/gsi/client" on:load={gisLoaded()}></script>
    <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/bootstrap-icons@1.11.2/font/bootstrap-icons.css">
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
            <ListGroupItem>Click and drag to enter your availability, X to delete</ListGroupItem>
            <ListGroupItem>Authorize Google to display your GCal events (these will not be sent)</ListGroupItem>
            <ListGroupItem>Indicate level of availability (definitely free or free if necessary) by clicking directly on the timeslot</ListGroupItem>
        </ListGroup>
    </Row>
    <Row>
        <!-- Change which buttons are shown based on the status of Google authorization -->
        <Col>
            {#if !googleLogin }
                <Button color="success" id="authorize_button" on:click={() => handleAuthClick()}>Authorize Google Account</Button>
            {/if}
            <!-- Allow user to choose to show Google calendar or hide it once they have logged in -->
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

<!-- Modal Component -->
<Modal isOpen={showEditEventModal} on:click={() => showEditEventModal=false}>
    <ModalHeader>
        Add Information about your Availability
    </ModalHeader>
    <!-- The body of the modal is radio buttons to edit the event -->
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

            Free if necessary
        </label>
        <Button color="success" on:click={() => updateEvent()}>Submit</Button>
    </ModalBody>
    <ModalFooter>
        <Button  on:click={() => showEditEventModal=false}>
            <Icon name="x-lg" />
        </Button>
    </ModalFooter>
</Modal>

<Calendar bind:this={ec} {plugins} {options} /> 
