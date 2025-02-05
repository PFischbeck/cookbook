<template>
    <div class="time">
        <button
            v-if="this.timer"
            type="button"
            :class="countdown === null ? 'icon-play' : 'icon-pause'"
            @click="timerToggle"
        ></button>
        <h4>{{ t("cookbook", label) }}</h4>
        <p>{{ displayTime }}</p>
    </div>
</template>

<script>
export default {
    name: "RecipeTimer",
    props: ["value", "phase", "label", "timer"],
    data() {
        return {
            countdown: null,
            hours: 0,
            minutes: 0,
            seconds: 0,
            showFullTime: false,
        }
    },
    watch: {
        value() {
            this.resetTimeDisplay()
        },
    },
    computed: {
        displayTime: function () {
            let text = ""
            if (this.showFullTime) {
                text += this.hours.toString().padStart(2, "0") + ":"
            } else {
                text += this.hours.toString() + ":"
            }
            text += this.minutes.toString().padStart(2, "0")
            if (this.showFullTime) {
                text += ":" + this.seconds.toString().padStart(2, "0")
            }
            return text
        },
    },
    methods: {
        onTimerEnd: function (button) {
            window.clearInterval(this.countdown)
            // I'll just use an alert until this functionality is finished
            let $this = this
            window.setTimeout(function () {
                // The short timeout is needed or Vue doesn't have time to update the countdown
                //  display to display 00:00:00
                alert(t("cookbook", "Cooking time is up!"))
                //cookbook.notify(t('cookbook', 'Cooking time is up!'))
                $this.countdown = null
                $this.showFullTime = false
                $this.resetTimeDisplay()
            }, 100)
        },
        resetTimeDisplay: function () {
            if (this.value.hours) {
                this.hours = parseInt(this.value.hours)
            } else {
                this.hours = 0
            }
            if (this.value.minutes) {
                this.minutes = parseInt(this.value.minutes)
            } else {
                this.minutes = 0
            }
            this.seconds = 0
        },
        timerToggle: function () {
            // We will switch to full time display the first time this method is invoked.
            // There should probably also be a way to reset the timer other than by letting
            //  it run its course...
            if (!this.showFullTime) {
                this.showFullTime = true
            }
            if (this.countdown === null) {
                // Pass this to callback function
                let $this = this
                this.countdown = window.setInterval(function () {
                    $this.seconds--
                    if ($this.seconds < 0) {
                        $this.seconds = 59
                        $this.minutes--
                    }
                    if ($this.minutes < 0) {
                        $this.minutes = 59
                        $this.hours--
                    }
                    if (
                        $this.hours === 0 &&
                        $this.minutes === 0 &&
                        $this.seconds === 0
                    ) {
                        $this.onTimerEnd()
                    }
                }, 1000)
            } else {
                window.clearInterval(this.countdown)
                this.countdown = null
            }
        },
    },
    mounted() {
        this.resetTimeDisplay()
    },
}
</script>

<style scoped>
.time {
    position: relative;
    flex-grow: 1;
    border: 1px solid var(--color-border-dark);
    margin: 1rem 2rem;
    border-radius: 3px;
    font-size: 1.2rem;
    text-align: center;
}
.time button {
    position: absolute;
    top: 0;
    left: 0;
    width: 36px;
    height: 36px;
    transform: translate(-50%, -50%);
}
.time h4 {
    padding: 0.5rem;
    border-bottom: 1px solid var(--color-border-dark);
    background-color: var(--color-background-dark);
    font-weight: bold;
}

.time p {
    padding: 0.5rem;
}

@media print {
    button {
        display: none !important;
    }
}
</style>
