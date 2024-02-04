<script>
    import Calendar from '@event-calendar/core';
    import TimeGrid from '@event-calendar/time-grid';
    import Interaction from '@event-calendar/interaction';
    import '@event-calendar/core/index.css';

    let plugins = [TimeGrid, Interaction];
    let eventsList = [];
    let ec;
    let options = {
        view: 'timeGridWeek',
        events: eventsList,
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
    }

//     const start = async () => {
//     // Initializes the client with the API key and the Translate API.
//     // @ts-ignore
//     gapi.client.init({
//       'apiKey': 'AIzaSyDDfNxkzJppzzegvfAr9WGc-Y0RzlquirU',
//       'discoveryDocs': ['https://www.googleapis.com/discovery/v1/apis/calendar/v3/rest'],
//     }).then(function() {
//       // Executes an API request, and returns a Promise.
//       // The method name `language.translations.list` comes from the API discovery.
//       return gapi.client.language.translations.list({
//         q: 'hello world',
//         source: 'en',
//         target: 'de',
//       });
//     }).then(function(response) {
//       console.log(response.result.data.translations[0].translatedText);
//     }, function(reason) {
//       console.log('Error: ' + reason.result.error.message);
//     });
//   };

//   const initializeGapi = async () => {
//     gapi.load('client', start);
//   }
</script>

<!-- <svelte:head>
    <script src="https://apis.google.com/js/api.js" on:load={initializeGapi}></script>
</svelte:head> -->
<Calendar bind:this={ec} {plugins} {options} />