<script>
    import Calendar from '@event-calendar/core';
    import TimeGrid from '@event-calendar/time-grid';
    import Interaction from '@event-calendar/interaction';
    import '@event-calendar/core/index.css';

    let plugins = [TimeGrid, Interaction];
    let googleEvents;
    let showingGoogle = false;
    let ec;
    let showAddEventModal = false;
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
        }
    };

    function addEvent(info) {
        ec.addEvent(info)
        ec = ec.unselect()
        showAddEventModal = true;
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

    function handleAuthClick() {
        tokenClient.callback = async (resp) => {
          if (resp.error !== undefined) {
            throw (resp);
          }
        //   document.getElementById('signout_button').style.visibility = 'visible';
        //   document.getElementById('authorize_button').innerText = 'Refresh';
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
    function handleSignoutClick() {
        const token = gapi.client.getToken();
        if (token !== null) {
          google.accounts.oauth2.revoke(token.access_token);
          gapi.client.setToken('');
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
            'maxResults': 10,
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
        // Flatten to string to display
        //console.log(events)
        googleEvents = events
    //     const output = events.reduce(
    //         (str, event) => `${str}${event.summary} (${event.start.dateTime || event.start.date})\n`,
    //         'Events:\n');
    //     console.log(output)
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
<button id="authorize_button" on:click={() => handleAuthClick()}>Authorize</button>
<button id="signout_button" on:click={() => handleSignoutClick()}>Sign Out</button>

{#if !showingGoogle }
    <button id="show Google calendar" on:click={() => showGoogleEvents()}>Show Google Calendar</button>
{:else}
    <button id="hide Google calendar" on:click={() => hideGoogleEvents()}>Hide Google Calendar</button>
{/if}

<Calendar bind:this={ec} {plugins} {options} />

<!-- Events:
ES 94 (2024-02-05T12:45:00-05:00)
CS 178 (2024-02-05T15:45:00-05:00)
Neuro 140 (2024-02-06T15:00:00-05:00)
ES 94 (2024-02-07T12:45:00-05:00)
CS 178 (2024-02-07T15:45:00-05:00)
CS 178 Section (2024-02-08T12:45:00-05:00)
DPI 664M (2024-02-08T16:30:00-05:00)
Neuro 140 Tutorial 2 (2024-02-08T18:00:00-05:00)
IC New Romantics (2024-02-08T19:00:00-05:00)
IC Alumni Dinner (2024-02-10T19:30:00-05:00) -->

<!-- {start: days[0] + " 00:00", end: days[0] + " 09:00", resourceId: 1, display: "background"}, -->