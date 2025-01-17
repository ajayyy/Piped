<template>
    <div class="uk-container uk-container-xlarge">
        <div data-shaka-player-container>
            <video
                data-shaka-player
                autoplay
                style="width: 100%; height: 100%"
            ></video>
        </div>
        <h1 class="uk-text-bold">{{ video.title }}</h1>

        <img :src="video.uploaderAvatar" loading="lazy" />
        <router-link class="uk-text-bold" v-bind:to="video.uploaderUrl || '/'">
            <a>{{ video.uploader }}</a>
        </router-link>

        <p class="uk-dark">
            <font-awesome-icon icon="thumbs-down"></font-awesome-icon>
            <b>{{ video.likes }}</b>
            &nbsp;
            <font-awesome-icon icon="thumbs-up"></font-awesome-icon>
            <b>{{ video.dislikes }}</b>
        </p>
        <p>
            <font-awesome-icon icon="eye"></font-awesome-icon>
            <b>{{ video.views }}</b> views
        </p>
        <p>
            Uploaded on <b>{{ video.uploadDate }}</b>
        </p>
        <a
            class="uk-button uk-button-small"
            style="background: #222"
            @click="showDesc = !showDesc"
        >
            {{ showDesc ? "+" : "-" }}
        </a>
        <p v-show="showDesc" class="uk-light" v-html="video.description"></p>
        <a v-if="sponsors && sponsors.segments"
            >Sponsors Segments: {{ sponsors.segments.length }}</a
        >

        <hr />

        <b>Auto Play next Video:</b>&nbsp;
        <input
            class="uk-checkbox"
            v-model="selectedAutoPlay"
            @change="onChange($event)"
            type="checkbox"
        />

        <div
            class="uk-tile-default uk-text-secondary"
            style="background: #0b0e0f; width: 300px"
            v-bind:key="related.url"
            v-for="related in video.relatedStreams"
        >
            <router-link class="uk-link-muted" v-bind:to="related.url">
                <p class="uk-text-emphasis">{{ related.title }}</p>
                <img
                    style="width: 100%"
                    v-bind:src="related.thumbnail"
                    loading="lazy"
                />
            </router-link>
            <p>
                <router-link
                    class="uk-link-muted"
                    v-bind:to="related.uploaderUrl || '/'"
                >
                    <p>{{ related.uploaderName }}</p>
                </router-link>
                <font-awesome-icon icon="eye"></font-awesome-icon>
                {{ related.views }} views
            </p>
        </div>
    </div>
</template>

<script>
import "shaka-player/dist/shaka-player.ui.js";
import("shaka-player/dist/controls.css");
const shaka = import("shaka-player/dist/shaka-player.ui.js");
import Constants from "@/Constants.js";

export default {
    name: "App",
    data() {
        return {
            video: {
                title: "Loading..."
            },
            sponsors: null,
            selectedAutoPlay: null,
            showDesc: true
        };
    },
    mounted() {
        this.selectedAutoPlay = Constants.AUTO_PLAY;
        this.getVideoData();
        this.getSponsors();
    },
    beforeUnmount() {
        if (window.player) {
            window.player.destroy();
            window.player = undefined;
            window.ui = undefined;
        }
    },
    watch: {
        "$route.query.v": function(v) {
            if (v) {
                this.getVideoData();
                this.getSponsors();
            }
        }
    },
    methods: {
        fetchVideo() {
            return this.fetchJson(
                Constants.BASE_URL + "/streams/" + this.$route.query.v
            );
        },
        async fetchSponsors() {
            return await this.fetchJson(
                Constants.BASE_URL +
                    "/sponsors/" +
                    this.$route.query.v +
                    "?category=" +
                    (localStorage && localStorage.getItem("selectedSkip")
                        ? encodeURIComponent(
                              '["' +
                                  localStorage
                                      .getItem("selectedSkip")
                                      .replace(",", '","') +
                                  '"]'
                          )
                        : encodeURIComponent(
                              '["sponsor", "interaction", "selfpromo", "music_offtopic"]'
                          ))
            );
        },
        onChange() {
            if (localStorage)
                localStorage.setItem("autoplay", this.selectedAutoPlay);
        },
        setPlayerAttrs(player, videoEl, dash, shaka) {
            player
                .load(
                    "data:application/dash+xml;charset=utf-8;base64," +
                        btoa(dash)
                )
                .then(() => {
                    this.video.subtitles.map(subtitle => {
                        player.addTextTrack(
                            subtitle.url,
                            "eng",
                            "SUBTITLE",
                            subtitle.mimeType,
                            null,
                            "English"
                        );
                        player.setTextTrackVisibility(true);
                    });
                    if (localStorage)
                        videoEl.volume = localStorage.getItem("volume") || 1;

                    const ui =
                        window.ui ||
                        (window.ui = new shaka.ui.Overlay(
                            player,
                            document.querySelector(
                                "div[data-shaka-player-container]"
                            ),
                            videoEl
                        ));
                    const config = {
                        overflowMenuButtons: [
                            "quality",
                            "captions",
                            "playback_rate"
                        ],
                        seekBarColors: {
                            base: "rgba(255, 255, 255, 0.3)",
                            buffered: "rgba(255, 255, 255, 0.54)",
                            played: "rgb(255, 0, 0)"
                        }
                    };
                    ui.configure(config);
                });
        },
        async getVideoData() {
            this.fetchVideo()
                .then(data => {
                    this.video = data;
                })
                .then(() => {
                    document.title = this.video.title + " - Piped";

                    this.video.description = this.video.description
                        .replaceAll("http://www.youtube.com", "")
                        .replaceAll("https://www.youtube.com", "")
                        .replaceAll("\n", "<br>");

                    const noPrevPlayer = !window.player;

                    var streams = [];

                    streams.push(...this.video.audioStreams);
                    streams.push(...this.video.videoStreams);

                    const dash = require("../utils/DashUtils.js").default.generate_dash_file_from_formats(
                        streams,
                        this.video.duration
                    );

                    const WatchVideo = this;

                    const videoEl = document.querySelector("video");

                    setTimeout(function() {
                        shaka
                            .then(shaka => shaka.default)
                            .then(shaka => {
                                if (noPrevPlayer) {
                                    shaka.polyfill.installAll();

                                    const player = new shaka.Player(videoEl);

                                    WatchVideo.player = player;
                                    window.player = player;

                                    WatchVideo.setPlayerAttrs(
                                        player,
                                        videoEl,
                                        dash,
                                        shaka
                                    );
                                } else {
                                    WatchVideo.setPlayerAttrs(
                                        window.player,
                                        videoEl,
                                        dash,
                                        shaka
                                    );
                                }
                            });
                    }, 0);

                    videoEl.setAttribute("poster", this.video.thumbnailUrl);

                    if (this.$route.query.t)
                        videoEl.currentTime = this.$route.query.t;

                    videoEl.addEventListener("loadedmetadata", function() {
                        const track = this.addTextTrack(
                            "captions",
                            "English",
                            "en"
                        );
                        track.mode = "showing";
                    });

                    if (noPrevPlayer) {
                        videoEl.addEventListener("timeupdate", () => {
                            if (
                                WatchVideo.sponsors &&
                                WatchVideo.sponsors.segments
                            ) {
                                const time = videoEl.currentTime;
                                WatchVideo.sponsors.segments.map(segment => {
                                    if (!segment.skipped) {
                                        const end = segment.segment[1];
                                        if (
                                            time >= segment.segment[0] &&
                                            time < end
                                        ) {
                                            console.log(
                                                "Skipped segment at " + time
                                            );
                                            videoEl.currentTime = end;
                                            segment.skipped = true;
                                            return;
                                        }
                                    }
                                });
                            }
                        });

                        videoEl.addEventListener("volumechange", () => {
                            if (localStorage)
                                localStorage.setItem("volume", videoEl.volume);
                        });

                        videoEl.addEventListener("ended", () => {
                            if (
                                WatchVideo.selectedAutoPlay &&
                                WatchVideo.video.relatedStreams.length > 0
                            )
                                WatchVideo.$router.push(
                                    WatchVideo.video.relatedStreams[0].url
                                );
                        });
                    }

                    //const parent = this.player.el().querySelector(".vjs-progress-holder")
                    //TODO: Add sponsors on seekbar: https://github.com/ajayyy/SponsorBlock/blob/e39de9fd852adb9196e0358ed827ad38d9933e29/src/js-components/previewBar.ts#L12
                });
        },
        async getSponsors() {
            if (!localStorage || localStorage.getItem("sponsorblock") !== false)
                this.fetchSponsors().then(data => (this.sponsors = data));
        }
    }
};
</script>
