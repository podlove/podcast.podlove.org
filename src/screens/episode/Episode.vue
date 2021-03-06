<template>
  <Layout>
    <episode-header
      :id="id"
      :title="`${episode.mnemonic} ${episode.title}`"
      :poster="episode.poster"
      :publication-date="episode.publicationDate"
      :duration="episode.duration"
      :contributors="contributors"
      ><episode-navigation :comments="hasComments" :shownotes="!!episode.content"
    /></episode-header>
    <div class="lg:w-full lg:flex lg:justify-center pt-20">
      <div class="lg:w-app">
        <section id="summary">
          <h2 class="font-mono inline-block border-gray-400 border-b-2 mb-6 mx-8 sm:mx-2">
            {{ $t('EPISODE.SUMMARY') }}
          </h2>
          <div class="font-light border-gray-400 border-b mb-12 pt-6 pb-12 px-6">
            {{ episode.summary }}
          </div>
        </section>
        <section id="shownotes" v-if="episode.content">
          <h2 class="font-mono inline-block border-gray-400 border-b-2 mb-6 mx-8 sm:mx-2">
            {{ $t('EPISODE.SHOWNOTES') }}
          </h2>
          <div
            class="font-light episode-content border-gray-400 border-b mb-12 pb-12 px-12"
            v-html="episode.content"
          ></div>
        </section>
        <section id="timeline">
          <h2 class="font-mono inline-block border-gray-400 border-b-2 mb-6 mx-8 sm:mx-2">
            {{ $t('EPISODE.TIMELINE') }}
          </h2>
          <timeline
            class="font-light border-gray-400 border-b mb-12 pb-12 pl-12 pr-6"
            :id="episode.id"
            :timeline="episode.timeline"
          />
        </section>
        <section id="discuss" v-if="hasComments">
          <discourse class="mb-12 px-2" />
        </section>
      </div>
    </div>
  </Layout>
</template>

<page-query>
query ($id: ID!) {
  podcastEpisode(id: $id) {
    id,
    mnemonic,
    path,
    title,
    summary,
    publicationDate,
    poster,
    duration,
    content,
    audio {
      url,
      size,
      title,
      mimeType
    },
    chapters {
      start,
      end,
      title,
      href,
      image
    },
    contributors {
      details {
        id,
        slug,
        name,
        nickname,
        avatar
      },
      role {
        title
      },
      group {
        title,
        slug
      }
    },
    timeline {
      node,
      title,
      type,
      start,
      end,
      texts {
        start,
        end,
        text
      },
      speaker {
        id,
        slug,
        avatar,
        name,
        nickname
      }
    }
  }
}
</page-query>

<static-query>
query {
  metadata {
    siteUrl,
    PodcastShow {
      title,
      poster
    }
  }
}
</static-query>

<script>
import { throttle } from 'throttle-debounce'
import scrollIntoView from 'scroll-into-view-if-needed'
import { mapActions, mapState } from 'redux-vuex'
import { compose, path, pathOr, propOr, prop, flatten } from 'ramda'
import { toPlayerTime, toHumanTime } from '@podlove/utils/time'

import { selectors } from '~/store/reducers'
import Timeline from '~/features/timeline/Timeline'
import Discourse from '~/features/comments/Discourse'
import { contributors, comments } from '~/config'

import EpisodeHeader from './Header'
import EpisodeNavigation from './Navigation'

export default {
  data: mapState({
    episodeId: selectors.current.episode,
    playtime: selectors.player.playtime,
    ghost: selectors.player.ghost.time,
    followContent: selectors.playbar.followContent
  }),

  components: {
    Timeline,
    EpisodeHeader,
    EpisodeNavigation,
    Discourse
  },

  computed: {
    id() {
      return path(['id'], this.episode)
    },
    follow() {
      return this.followContent && this.episodeId === this.id
    },
    episode() {
      return path(['podcastEpisode'], this.$page)
    },
    contributors() {
      return pathOr([], ['contributors'], this.episode)
        .filter((contributor) => contributors.groups.includes(path(['group', 'slug'], contributor)))
        .map((contributor) => ({
          ...propOr({}, 'details', contributor),
          group: path(['group', 'title'], contributor),
          role: path(['role', 'title'], contributor)
        }))
    },
    hasComments() {
      return !!comments.discourse
    }
  },

  mounted() {
    if (this.followContent) {
      this.scroll()
    }
  },

  watch: {
    playtime() {
      this.scroll()
    },
    ghost() {
      this.scroll()
    },
    followContent() {
      this.followContent && this.scroll()
    }
  },

  methods: {
    ...mapActions('playEpisode'),

    scroll() {
      if (!this.follow) {
        return
      }

      this.scrollIntoView()
    },
    scrollIntoView: throttle(500, () => {
      const node =
        document.getElementById('transcript-ghost-active') ||
        document.getElementById('transcript-active')

      return (
        node &&
        scrollIntoView(node, {
          behavior: 'smooth',
          scrollMode: 'always',
          block: 'center',
          inline: 'center'
        })
      )
    }),
    duration: compose(toHumanTime, toPlayerTime)
  },

  metaInfo() {
    const metadata = path(['$static', 'metadata'], this)
    const authors = propOr([], 'contributors', this)
    const audio = propOr([], 'audio', this.episode)
    const poster = prop('poster', this.episode) || path(['PodloveShow', 'poster'], this.episode)

    return {
      title: prop('title', this.episode),
      meta: [
        {
          name: 'description',
          content: prop('summary', this.episode)
        },
        // Authors
        ...authors.map((author) => ({
          name: 'author',
          content: prop('name', author)
        })),
        // Open Graph
        {
          property: 'og:type',
          content: 'website'
        },
        {
          property: 'og:site_name',
          content: path(['PodcastShow', 'title'], metadata)
        },
        {
          property: 'og:title',
          content: prop('title', this.episode)
        },
        {
          property: 'og:url',
          content: `${prop('siteUrl', metadata)}${prop('path', this.episode)}`
        },
        {
          property: 'og:description',
          content: prop('summary', this.episode)
        },
        ...(poster
          ? [
              {
                property: 'og:image',
                content: prop('siteUrl', metadata) + '/assets/images/' + poster
              }
            ]
          : []),
        ...flatten(
          audio.map((src) => [
            {
              property: 'og:audio:type',
              content: src.mimeType
            },
            {
              property: 'og:audio',
              content: src.url
            }
          ])
        )
      ]
    }
  }
}
</script>

<style>
.episode-content ul {
  list-style-type: disc !important;
  margin-bottom: 2em;
}

.episode-content li {
  margin-bottom: 0.5em;
}

.episode-content h1 {
  font-weight: bold;
  font-size: 1.5em;
  margin-left: -1.5em;
  margin-bottom: 1em;
}

.episode-content h2 {
  font-weight: bold;
  font-size: 1.2em;
  margin-left: -1.5em;
  margin-bottom: 0.75em;
}

.episode-content h3 {
  font-weight: bold;
  font-size: 1em;
  margin-left: -1.5em;
  margin-bottom: 0.75em;
}

.episode-content a {
  border-bottom: 1px solid rgba(203, 213, 224);
  padding-bottom: 1px;
}
</style>
