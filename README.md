<a name="readme-top"></a>

# Mapty App Project

## Table of Contents

- [Project-Description](#Project-Description)
- [Prerequisites](#Prerequisites)
- [Installing](#Installing)
- [Steps](#Steps)
- [Author](#Author)
- [Credits](#Credits)

## Project-Description

The starter project files had the page structure and interface set in HTML and CSS for the Mapty app. I had to implement a workout tracker that contains the location and activity details on the map using Leaflet library API and maintain the data in `localStorage` browser API.

## Prerequisites

Any code editor (e.g: VSCode, Atom,... etc)

## Installing

Terminal commands to start using the project:

- Get a copy on your machine

```
`git clone https://github.com/Aragorn-Elessar/Mapty.git`
```

- Call into the directory location

```
`cd Mapty`
```

- Opens code in `VSCode`

```
`code .`
```

<p align="right">(<a href="#readme-top">back to top</a>)</p>

## Steps

### Main Application Architecture

- Define selectors for needed elements

```js
const form = document.querySelector('.form');
const containerWorkouts = document.querySelector('.workouts');
const inputType = document.querySelector('.form__input--type');
const inputDistance = document.querySelector('.form__input--distance');
const inputDuration = document.querySelector('.form__input--duration');
const inputCadence = document.querySelector('.form__input--cadence');
const inputElevation = document.querySelector('.form__input--elevation');
```

- Private variables defined in the class field

```js
class App {
  #map;
  #mapZoomLevel = 13;
  #mapEvent;
  #workouts = [];
```

- Functions that needs to be run on page load and attached listeners for main events

```js
  constructor() {
    // Ger user's position
    this._getPosition();

    // Get data from local storage
    this._getLocalStorage();

    // Attach event handlers
    // bind this to app object instead of form element
    form.addEventListener('submit', this._newWorkout.bind(this));

    inputType.addEventListener('change', this._toggleElevationfield);

    containerWorkouts.addEventListener('click', this._moveToPopup.bind(this));
  }
```

- Get current user position and pass it to `loadMap` function

```js
  _getPosition() {
    if (navigator.geolocation)
      // Execute callback function & pass position as soon as current user position determined
      navigator.geolocation.getCurrentPosition(
        // bind this to app object instead of undefined
        this._loadMap.bind(this),
        function () {
          alert('Could not get your position');
        }
      );
  }
```

- Load map view with the user passed coords and specified zoom level, and attach a listener for map clicks to show the workout form

```js
  _loadMap(position) {
    const { latitude } = position.coords;
    const { longitude } = position.coords;
    console.log(`https://www.google.com/maps/@${latitude},${longitude}`);

    const coords = [latitude, longitude];

    this.#map = L.map('map').setView(coords, this.#mapZoomLevel);

    L.tileLayer('https://tile.openstreetmap.fr/hot/{z}/{x}/{y}.png', {
      attribution:
        '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors',
    }).addTo(this.#map);

    // Handling clicks on map
    this.#map.on('click', this._showForm.bind(this));

    // Render marker for localStorage after map is available
    this.#workouts.forEach(work => {
      this._renderWorkoutMarker(work);
    });
  }
```

- Show workout form and and set `#mapEvent` value to the clicked position event

```js
  _showForm(mapE) {
    this.#mapEvent = mapE;

    // Show form & focus on input field
    form.classList.remove('hidden');
    inputDistance.focus();
  }
```

- Empty inputs and hide form after the user submits the workout, also reassign the display grid value after the form is hidden to restore the animation using `setTimeout()` method

```js
  _hideForm() {
    // Empty inputs
    inputDistance.value =
      inputDuration.value =
      inputCadence.value =
      inputElevation.value =
        '';

    // Prevent animation in form class
    form.style.display = 'none';
    form.classList.add('hidden');
    // Add grid value to restore display features
    setTimeout(() => (form.style.display = 'grid'), 1000);
  }
```

- Toggle elevation/cadence field for cycling/running workouts

```js
  _toggleElevationfield() {
    inputElevation.closest('.form__row').classList.toggle('form__row--hidden');
    inputCadence.closest('.form__row').classList.toggle('form__row--hidden');
  }
```

- Create and render a new running/cycling workout on map & list depending on the validated user data entered, and save it in local storage

```js
  _newWorkout(e) {
    const validInputs = (...inputs) => {
      return inputs.every(inp => Number.isFinite(inp));
    };
    const allPositive = (...inputs) => {
      return inputs.every(inp => inp > 0);
    };

    // Prevent page from reloading
    e.preventDefault();

    // Get data from form
    const type = inputType.value;
    // + converts the string to a number
    const distance = +inputDistance.value;
    const duration = +inputDuration.value;
    const { lat, lng } = this.#mapEvent.latlng;
    let workout;

    // If activity running, create running object
    if (type === 'running') {
      const cadence = +inputCadence.value;

      // Check if data is valid
      if (
        !validInputs(distance, duration, cadence) ||
        !allPositive(distance, duration, cadence)
      )
        return alert('Inputs have to be positive numbers!');

      // Create new workout
      workout = new Running([lat, lng], distance, duration, cadence);
    }

    // If activity cycling, create cycling object
    if (type === 'cycling') {
      const elevation = +inputElevation.value;

      // Check if data is valid
      if (
        !validInputs(distance, duration, elevation) ||
        !allPositive(distance, duration)
      )
        return alert('Inputs have to be positive numbers!');

      // Create new workout
      workout = new Cycling([lat, lng], distance, duration, elevation);
    }

    // Add new object to workouts array
    this.#workouts.push(workout);

    // Render workout on list
    this._renderWorkoutMarker(workout);

    // Render workout on list
    this._renderWorkout(workout);

    // Hide form + clear input field
    this._hideForm(workout);

    // Set local storage to all workouts
    this._setLocalStorage();
  }
```

- Render the workout marker using the passed coordinates, select the popup style and set its content to the workout description

```js
  // Render workout on map as marker
  _renderWorkoutMarker(workout) {
    L.marker(workout.coords)
      .addTo(this.#map)
      .bindPopup(
        L.popup({
          maxWidth: 250,
          minWidth: 100,
          autoClose: false,
          closeButton: false,
          closeOnClick: false,
          className: `${workout.type}-popup`,
        })
      )
      .setPopupContent(
        `${workout.type === 'running' ? '🏃‍♂️' : '🚴‍♂️'} ${workout.description}`
      )
      .openPopup();
  }
```

- Create & insert the new workout in the app form list depending on its type using ternary operators

```js
  _renderWorkout(workout) {
    const html = `
    <li class="workout workout--${workout.type}"   data-id="${workout.id}">
      <h2 class="workout__title">${workout.description}</h2>
      <div class="workout__details">
        <span class="workout__icon">${
          workout.type === 'running' ? '🏃‍♂️' : '🚴‍♂️'
        }</span>
        <span class="workout__value">${workout.distance}</span>
        <span class="workout__unit">km</span>
      </div>
      <div class="workout__details">
        <span class="workout__icon">⏱</span>
        <span class="workout__value">${workout.duration}</span>
        <span class="workout__unit">min</span>
      </div>${
        workout.type === 'running'
          ? `
      <div class="workout__details">
        <span class="workout__icon">⚡️</span>
        <span class="workout__value">${workout.pace.toFixed(1)}</span>
        <span class="workout__unit">min/km</span>
      </div>
      <div class="workout__details">
        <span class="workout__icon">🦶🏼</span>
        <span class="workout__value">${workout.cadence}</span>
        <span class="workout__unit">spm</span>
      </div>
  `
          : `
      <div class="workout__details">
        <span class="workout__icon">⚡️</span>
        <span class="workout__value">${workout.speed.toFixed(1)}</span>
        <span class="workout__unit">km/h</span>
      </div>
      <div class="workout__details">
        <span class="workout__icon">⛰</span>
        <span class="workout__value">${workout.elevation}</span>
        <span class="workout__unit">m</span>
      </div>
          `
      }</li>
    `;

    form.insertAdjacentHTML('afterend', html);
  }
```

- Move to the map marker for the clicked workout in the list

```js
  _moveToPopup(e) {
    const workoutEl = e.target.closest('.workout');

    // Guard clause
    if (!workoutEl) return;

    const workout = this.#workouts.find(
      work => work.id === workoutEl.dataset.id
    );

    this.#map.setView(workout.coords, this.#mapZoomLevel, {
      animate: true,
      pan: {
        duration: 1,
      },
    });
  }
```

- Store workouts in `localStorage` browser API

```js
  _setLocalStorage() {
    localStorage.setItem('workouts', JSON.stringify(this.#workouts));
  }
```

- Retrieve and render workouts data from the local storage

```js
  _getLocalStorage() {
    const data = JSON.parse(localStorage.getItem('workouts'));

    // Guard clause
    if (!data) return;

    this.#workouts = data;

    this.#workouts.forEach(work => {
      this._renderWorkout(work);
    });
  }
```

- Reset method to clear local storage

```js
  reset() {
    localStorage.removeItem('workouts');
    location.reload();
  }
}
```

### Application Sub Classes

- A `Workout` class for shared data and a method to set the description for `Running` & `Cycling` classes

```js
class Workout {
  date = new Date();
  id = (Date.now() + '').slice(-10);
  clicks = 0;

  constructor(coords, distance, duration) {
    this.coords = coords; // [lat, lng]
    this.distance = distance; // in km
    this.duration = duration; // in min
  }

  _setDescription() {
    // prettier-ignore
    const months = ['January', 'February', 'March', 'April', 'May', 'June', 'July', 'August', 'September', 'October', 'November', 'December'];

    this.description = `${this.type[0].toUpperCase() + this.type.slice(1)} on ${
      months[this.date.getMonth()]
    } ${this.date.getDate()}`;
  }
}
```

- Class for running workout data with a call to `calcPace()` method which calculates the pace and `setDescription()` method as well

```js
class Running extends Workout {
  type = 'running';

  constructor(coords, distance, duration, cadence) {
    super(coords, distance, duration);
    this.cadence = cadence;
    this.calcPace();
    this._setDescription();
  }

  calcPace() {
    // min/km
    this.pace = this.duration / this.distance;
    return this.pace;
  }
}
```

- Class for cycling workout data with a call to `calcSpeed()` method which calculates the speed and `setDescription()` method too

```js
class Cycling extends Workout {
  type = 'cycling';

  constructor(coords, distance, duration, elevationGain) {
    super(coords, distance, duration);
    this.elevation = elevationGain;
    this.calcSpeed();
    this._setDescription();
  }

  calcSpeed() {
    // km/h
    this.speed = this.distance / (this.duration / 60);
    return this.speed;
  }
}
```

## Author

[Mahmoud Gadallah](https://github.com/Aragorn-Elessar)

## Credits

A [Udemy](https://www.udemy.com) project, provided by [Jonas Schmedtmann](https://www.udemy.com/user/jonasschmedtmann/) JavaScript course

<p align="right">(<a href="#readme-top">back to top</a>)</p>
