Define Variables
```
const PROWLER_API_KEY = 'YOUR_API_KEY_HERE';
const PROWLER_URL = 'YOUR_URL_HERE';
```

Access the API using browser

```
fetch(`https://${PROWLER_URL}/api/v1/tenants`, {
  method: 'GET',
  headers: {
    'Authorization': `Api-Key ${PROWLER_API_KEY}`,
    'Content-Type': 'application/vnd.api+json'
  }
})
.then(response => {
  if (!response.ok) {
    throw new Error(`HTTP error! Status: ${response.status}`);
  }
  return response.json();
})
.then(data => {
  console.log('Success:', data);
})
.catch(error => {
  console.error('Error fetching data:', error);
});
```
