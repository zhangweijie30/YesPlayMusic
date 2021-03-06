<template>
  <div class="artist" v-show="show">
    <div class="artist-info">
      <div class="head">
        <img :src="artist.img1v1Url | resizeImage(1024)" />
      </div>
      <div>
        <div class="name">{{ artist.name }}</div>
        <div class="artist">Artist</div>
        <div class="statistics">
          {{ artist.musicSize }} Songs · {{ artist.albumSize }} Albums ·
          {{ artist.mvSize }} Music Videos
        </div>
        <div class="buttons">
          <ButtonTwoTone @click.native="playPopularSongs()" :iconClass="`play`">
            {{ $t('play') }}
          </ButtonTwoTone>
        </div>
      </div>
    </div>
    <div class="latest-release">
      <div class="section-title">{{ $t('artist.latestRelease') }}</div>
      <div class="release">
        <div class="container">
          <Cover
            :url="latestRelease.picUrl | resizeImage"
            :showPlayButton="true"
            :showBlackShadow="true"
            :type="`album`"
            :id="latestRelease.id"
            :hoverEffect="true"
            :size="128"
            :playButtonSize="36"
            :shadowMargin="8"
            :radius="8"
          />
          <div class="info">
            <div class="name">
              <router-link :to="`/album/${latestRelease.id}`">{{
                latestRelease.name
              }}</router-link>
            </div>
            <div class="date">
              {{ latestRelease.publishTime | formatDate }}
            </div>
            <div class="type">
              {{ latestRelease.type | formatAlbumType(latestRelease) }} ·
              {{ latestRelease.size }} Songs
            </div>
          </div>
        </div>
        <div></div>
      </div>
    </div>
    <div class="popular-tracks">
      <div class="section-title">{{ $t('artist.popularSongs') }}</div>
      <TrackList
        :tracks="popularTracks.slice(0, showMorePopTracks ? 24 : 12)"
        :type="'tracklist'"
      />

      <div class="show-more">
        <button @click="showMorePopTracks = !showMorePopTracks">
          <span v-show="!showMorePopTracks">{{ $t('artist.showMore') }}</span>
          <span v-show="showMorePopTracks">{{ $t('artist.showLess') }}</span>
        </button>
      </div>
    </div>
    <div class="albums" v-if="albums.length !== 0">
      <div class="section-title">{{ $t('artist.album') }}</div>
      <CoverRow
        :type="'album'"
        :items="albums"
        :subText="'releaseYear'"
        :showPlayButton="true"
      />
    </div>
    <div class="mvs" v-if="mvs.length !== 0">
      <div class="section-title">MVs</div>
      <MvRow :mvs="mvs" subtitle="publishTime" />
    </div>
    <div class="eps">
      <div class="section-title">{{ $t('artist.EPSingle') }}</div>
      <CoverRow
        :type="'album'"
        :items="eps"
        :subText="'albumType+releaseYear'"
        :showPlayButton="true"
      />
    </div>
  </div>
</template>

<script>
import { mapMutations, mapActions, mapState } from "vuex";
import { getArtist, getArtistAlbum, artistMv } from "@/api/artist";
import { mapTrackPlayableStatus } from "@/utils/common";
import { playAList } from "@/utils/play";
import NProgress from "nprogress";

import ButtonTwoTone from "@/components/ButtonTwoTone.vue";
import TrackList from "@/components/TrackList.vue";
import CoverRow from "@/components/CoverRow.vue";
import Cover from "@/components/Cover.vue";
import MvRow from "@/components/MvRow.vue";

export default {
  name: "Artist",
  components: { Cover, ButtonTwoTone, TrackList, CoverRow, MvRow },
  data() {
    return {
      show: false,
      artist: {
        img1v1Url:
          "https://p1.music.126.net/VnZiScyynLG7atLIZ2YPkw==/18686200114669622.jpg",
      },
      popularTracks: [],
      albumsData: [],
      latestRelease: {
        picUrl: "",
        publishTime: 0,
        id: 0,
        name: "",
        type: "",
        size: "",
      },
      showMorePopTracks: false,
      mvs: [],
    };
  },
  computed: {
    ...mapState(["player"]),
    albums() {
      return this.albumsData.filter((a) => a.type === "专辑");
    },
    eps() {
      return this.albumsData.filter((a) =>
        ["EP/Single", "EP", "Single"].includes(a.type)
      );
    },
  },
  methods: {
    ...mapMutations(["appendTrackToPlayerList"]),
    ...mapActions(["playFirstTrackOnList", "playTrackOnListByID"]),
    loadData(id, next = undefined) {
      getArtist(id).then((data) => {
        this.artist = data.artist;
        this.popularTracks = data.hotSongs;
        this.popularTracks = mapTrackPlayableStatus(this.popularTracks);
        if (next !== undefined) next();
        NProgress.done();
        this.show = true;
      });
      getArtistAlbum({ id: id, limit: 200 }).then((data) => {
        this.albumsData = data.hotAlbums;
        this.latestRelease = data.hotAlbums[0];
      });
      artistMv(id).then((data) => {
        this.mvs = data.mvs;
      });
    },
    goToAlbum(id) {
      this.$router.push({
        name: "album",
        params: { id },
      });
    },
    playPopularSongs(trackID = "first") {
      let trackIDs = this.popularTracks.map((t) => t.id);
      playAList(trackIDs, this.artist.id, "artist", trackID);
    },
  },
  created() {
    this.loadData(this.$route.params.id);
  },
  activated() {
    if (this.show) {
      if (this.artist.id.toString() !== this.$route.params.id) {
        this.show = false;
        NProgress.start();
        this.loadData(this.$route.params.id);
      }
    }
  },
  beforeRouteUpdate(to, from, next) {
    NProgress.start();
    this.artist.img1v1Url =
      "https://p1.music.126.net/VnZiScyynLG7atLIZ2YPkw==/18686200114669622.jpg";
    this.loadData(to.params.id, next);
  },
};
</script>

<style lang="scss" scoped>
.artist-info {
  display: flex;
  align-items: center;
  margin-bottom: 72px;
  img {
    height: 192px;
    width: 192px;
    border-radius: 50%;
    margin-right: 56px;
    box-shadow: rgba(0, 0, 0, 0.2) 0px 12px 16px -8px;
  }
  .name {
    font-size: 56px;
    font-weight: 700;
  }

  .artist {
    font-size: 18px;
    color: rgba(0, 0, 0, 0.88);
    margin-top: 24px;
  }

  .statistics {
    font-size: 14px;
    color: rgba(0, 0, 0, 0.68);
    margin-top: 2px;
  }

  .buttons {
    margin-top: 26px;
    display: flex;
    .shuffle {
      padding: 8px 11px;
      .svg-icon {
        margin: 0;
      }
    }
  }
}

.section-title {
  font-weight: 600;
  font-size: 22px;
  color: rgba(0, 0, 0, 0.88);
  margin-bottom: 16px;
  margin-top: 46px;
}

.latest-release {
  .release {
    display: flex;
  }
  .container {
    display: flex;
    align-items: center;
    border-radius: 12px;
  }
  img {
    height: 96px;
    border-radius: 8px;
  }
  .info {
    margin-left: 24px;
  }
  .name {
    font-size: 18px;
    font-weight: 600;
    color: rgba(0, 0, 0, 0.88);
    margin-bottom: 8px;
  }
  .date {
    font-size: 14px;
    color: rgba(0, 0, 0, 0.78);
  }
  .type {
    margin-top: 2px;
    font-size: 12px;
    color: rgba(0, 0, 0, 0.68);
  }
}

.popular-tracks {
  .show-more {
    display: flex;

    button {
      padding: 4px 8px;
      margin-top: 8px;
      border-radius: 6px;
      font-size: 12px;
      color: rgba(0, 0, 0, 0.78);
      font-weight: 600;
      &:hover {
        background: #f5f5f7;
        color: rgba(0, 0, 0, 0.96);
      }
    }
  }
}
</style>
