<script>
    import '@event-calendar/core/index.css';
    import { goto } from '$app/navigation';
    import { nameStore } from './stores.js';
    import { Button, Input, Container, Col } from '@sveltestrap/sveltestrap';

    const CLIENT_ID = '669688591392-rhdn9ebnpq24fc3l08m45ud6tbh8rf4j.apps.googleusercontent.com';
    const API_KEY = 'AIzaSyDDfNxkzJppzzegvfAr9WGc-Y0RzlquirU';

    const DISCOVERY_DOC = 'https://www.googleapis.com/discovery/v1/apis/calendar/v3/rest';
    const SCOPES = 'https://www.googleapis.com/auth/calendar.readonly';
    let tokenClient;
    let gapiInited = false;
    let gisInited = false;
    let name;

    //Initializing Google API, logging in with credentials
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

    // When a user clicks submit they will be redirected to the calendar page
    function login() {
        nameStore.set(name);
        goto('/calendar')
    }

</script>

<svelte:head>
    <script async defer src="https://apis.google.com/js/api.js" on:load={gapiLoaded()}></script>
    <script async defer src="https://accounts.google.com/gsi/client" on:load={gisLoaded()}></script>
</svelte:head>

<Container>
  <Col xs="auto">
      <h1>Event: Weekly Board Meeting</h1>
      <h3>Enter your name below</h3>
  </Col>
</Container>

<Container>
    <Input placeholder="Name" bind:value={name}/>
    <Button color="primary" id="login" on:click={() => login()}>Submit</Button>
</Container>

<!-- CITATIONS: -->
<!-- For Google Calendar API:
- https://developers.google.com/calendar/api/quickstart/js
- https://www.appsloveworld.com/google-api/4/how-to-load-google-api-client-library-with-sveltekit
- https://stackoverflow.com/questions/70570231/how-to-load-google-api-client-library-with-sveltekit -->
<!-- For Calendar Package:
- https://github.com/vkurko/calendar
For Sveltestrap:
- https://sveltestrap.js.org/?path=/docs/sveltestrap-overview--docs -->

<!-- BREAKDOWN OF WORK: -->
<!-- Kara
- Google API integration
- Editing Calendar / changing degree of availability
- Modals
- Styling
Hever:
- Initial calendar / github setup
- Adding intro page + store
- Styling page 1 -->
