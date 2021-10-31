<script lang="ts">
    type ShiftedTime = { time: string; ampm: string, date: string };

    const SHIFT_STEP_MINUTES = 15;

    let shiftPretty = "·";
    let shiftMinutes = 0;  // shiftRaw converted to a ladder of SHIFT_STEP_MINUTES
    let shiftRaw = 0;

    let currentCities = [
        { name: "San Francisco", tzCode: "America/Los_Angeles", time: {time: "", ampm: "", date: ""} },
        { name: "Almaty", tzCode: "Asia/Almaty", time: {time: "", ampm: "", date: ""} },
        { name: "New York", tzCode: "America/New_York", time: {time: "", ampm: "", date: ""} },
    ];

    function shiftDateTime(date: Date) : Date {
        if (shiftMinutes == 0) return date;
        // Divide into quarters: 0, 15, 30, 45
        if (0 < shiftMinutes) {
            let steps = Math.floor(shiftMinutes / SHIFT_STEP_MINUTES);
            let nextQuarterMinutes = Math.floor(date.getMinutes() / 15) * 15 + 15 * steps;
            let smallShiftMinutes = nextQuarterMinutes - date.getMinutes();
            return new Date(date.getTime() + smallShiftMinutes * 60 * 1000);
        }
        let steps = Math.floor(Math.abs(shiftMinutes) / SHIFT_STEP_MINUTES) - 1;
        let prevQuarterMinutes = Math.floor(date.getMinutes() / 15) * 15 - 15 * steps;
        let smallShiftMinutes = date.getMinutes() - prevQuarterMinutes;
        return new Date(date.getTime() - smallShiftMinutes * 60 * 1000);
    }

    function getShiftedTimeInTimeZone(tzCode: string) : ShiftedTime {
        let now = new Date();
        let shiftedNow = shiftDateTime(now);
        let formatter = new Intl.DateTimeFormat("en-US", {
            year: "2-digit", month: "2-digit", day: "2-digit",
            hour: '2-digit', minute: '2-digit', hour12: true,
            timeZoneName: 'short',
            timeZone: tzCode,
        });
        let formatterLocal = new Intl.DateTimeFormat("en-US", {
            year: "2-digit", month: "2-digit", day: "2-digit",
        });
        const rtf = new Intl.RelativeTimeFormat("en-US", {
            numeric: "auto",
            style: "long",
        });
        let shiftedTzNowStr = formatter.format(shiftedNow);
        let nowStrLocalTz = formatterLocal.format(now);
        let diffDaysFromLocal = (new Date(shiftedTzNowStr.substr(0, 8)).valueOf() - new Date(nowStrLocalTz.substr(0, 8)).valueOf()) / 1000 / 60 / 60 / 24;
        return {
            time: shiftedTzNowStr.substr(10, 5),
            ampm: shiftedTzNowStr.substr(16, 2),
            date: rtf.format(diffDaysFromLocal, "day"),
        };
    }

    function calculateCurrentCityTimes() {
        let newCities = [];
        for (let city of currentCities) {
            const cityNow = getShiftedTimeInTimeZone(city.tzCode);
            newCities.push(Object.assign({}, city, {
                time: cityNow
            }));
        }
        currentCities = newCities;
    }

    function calculateCurrentShift() {
        shiftMinutes = Math.floor(shiftRaw / SHIFT_STEP_MINUTES) * SHIFT_STEP_MINUTES;

        let absShift = Math.abs(shiftMinutes);
        let hour = Math.floor(absShift / 60);
        let minute = `${absShift % 60}`;
        if (minute.length == 1) minute = `0${minute}`;

        if (shiftMinutes === 0) {
            shiftPretty = "·";
        } else if (shiftMinutes > 0) {
            shiftPretty = `+${hour}:${minute}`;
        } else {
            shiftPretty = `-${hour}:${minute}`;
        }

        calculateCurrentCityTimes();
    }

    function handleMouseWheel(e: WheelEvent) {
        if (e.type !== 'mousewheel') return;
        if (e.deltaX === 0) return;
        shiftRaw = shiftRaw - e.deltaX * 0.9;
        calculateCurrentShift();
    }

    function resetShift(e: Event) {
        shiftRaw = 0;
        calculateCurrentShift();
    }

    calculateCurrentCityTimes();
    setInterval(calculateCurrentCityTimes, 10 * 1000);
</script>

<svelte:window on:mousewheel={handleMouseWheel} />

<main>
    <h1>Time Zone Converter</h1>
    <div class="converter">
        <div class="time-control">
            {#if shiftMinutes !== 0}
                <div class="time-shift" on:click={resetShift}>
                    <span>{shiftPretty}</span>
                    <span class="reset">×</span>
                </div>
            {:else}
                <div class="time-shift-disabled">•</div>
            {/if}
            <div class="slider" style="background-position-x: {shiftMinutes * 0.5}px"></div>
        </div>
        {#each currentCities as city }
            <div class="timezone">
                <div class="tz-name">{city.name}</div>
                <div class="tz-right-side">
                    <div class="tz-time">
                        <span class="time-value">{city.time.time}</span>
                        <span class="ampm">{city.time.ampm}</span>
                    </div>
                    <div class="tz-date">{city.time.date}</div>
                </div>
            </div>
        {/each}
    </div>
</main>

<style>
    main {
        text-align: center;
        padding: 1em;
        margin: 0 auto;
        background-color: red;
    }

    h1 {
    font-size: 4em;
        font-weight: 100;
    }

    .converter {
        background-color: #3E444F;
        width: 400px;
        margin: 0 auto;
        display: flex;
        flex-direction: column;
        border-radius: 5px;
    }

    .converter > .timezone {
        display: flex;
        flex-wrap: wrap;
        align-content: space-between;
        justify-content: center;
        padding: 25px 30px;
        align-items: center;
        border-bottom: 0.5px solid #888;
    }

    .converter > .timezone:last-child {
        border-bottom: none;
    }

    .time-control {
        display: flex;
        flex-wrap: wrap;
        align-content: space-between;
        justify-content: center;
        align-items: center;
        border-bottom: 0.5px solid #888;
        padding: 15px 0 24px 0;
    }

    .time-shift {
        color: #2D333E;
        background-color: #EEEEEE;
        border-radius: 8px;
        padding: 2px 8px;
        font-size: 0.9em;
    }

    .time-shift:hover {
        cursor: pointer;
        background-color: #FFF;
        color: #424853;
    }

    .time-shift-disabled {
        color: #424853;
        background-color: #AAA;
        border-radius: 12px;
        padding: 2px 8px;
        font-size: 0.9em;
        cursor: default;
    }

    .time-shift > .reset {
        font-weight: bold;
    }

    .slider {
        background-image: url(/slider-dark.svg);
        width: 100%;
        overflow: hidden;
        height: 49px;
        opacity: 0.3;
        background-size: 49px 73px;
    }

    .tz-name {
        text-align: center;
        vertical-align: middle;
        font-size: 1.5em;
    }

    .tz-right-side {
        margin-left: auto;
        font-size: 1.5em;
    }

    .tz-right-side > .tz-time {
        width: 96px;
        display: flex;
        padding: 0;
        flex-direction: row;
    }

    .tz-right-side > .tz-time > .time-value {
        width: 69px;
        text-align: left;
    }

    .tz-right-side .ampm {
        font-size: 0.5em;
        vertical-align: top;
    }

    .tz-right-side .tz-date {
        text-align: left;
        font-size: 0.5em;
        color: #BBBBBB;
    }

    @media (min-width: 640px) {
        main {
            background-color: #2D333E;
        }
    }
</style>
