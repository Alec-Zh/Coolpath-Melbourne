<script setup>
import { ref, onMounted } from 'vue'
import { useRouter } from 'vue-router'
import NavBar from '../components/NavBar.vue'
import Footer from '../components/Footer.vue'
// import './codex-homepage.css'
const router = useRouter()

const expandedCards = ref([true, true, true, true])

const toggleCard = (index) => {
  expandedCards.value[index] = !expandedCards.value[index]
}

// Scroll animation
onMounted(() => {
  const animEls = document.querySelectorAll('[data-animate]')
  const animObs = new IntersectionObserver(
    (entries) => {
      entries.forEach((e) => {
        if (e.isIntersecting) {
          e.target.classList.add('is-visible')
          animObs.unobserve(e.target)
        }
      })
    },
    { threshold: 0.08, rootMargin: '0px 0px -50px 0px' },
  )
  animEls.forEach((el) => animObs.observe(el))

  // Stagger children
  document.querySelectorAll('.mh-stagger').forEach((container) => {
    new IntersectionObserver(
      (entries) => {
        entries.forEach((e) => {
          if (!e.isIntersecting) return
          Array.from(e.target.children).forEach((c, i) =>
            setTimeout(() => c.classList.add('is-visible'), i * 130),
          )
        })
      },
      { threshold: 0.05 },
    ).observe(container)
  })

  // Count-up
  document.querySelectorAll('.mh-count').forEach((el) => {
    new IntersectionObserver(
      (entries) => {
        entries.forEach((e) => {
          if (!e.isIntersecting) return
          const target = +el.dataset.target
          let cur = 0
          const t = setInterval(() => {
            cur = Math.min(cur + target / 40, target)
            el.textContent = Math.round(cur)
            if (cur >= target) clearInterval(t)
          }, 35)
        })
      },
      { threshold: 0.5 },
    ).observe(el)
  })

  // Interactive impact cards
  document.querySelectorAll('.mh-icard').forEach((card) => {
    card.addEventListener('pointermove', (event) => {
      const rect = card.getBoundingClientRect()
      card.style.setProperty('--mx', `${event.clientX - rect.left}px`)
      card.style.setProperty('--my', `${event.clientY - rect.top}px`)
    })
    card.addEventListener('pointerleave', () => {
      card.style.removeProperty('--mx')
      card.style.removeProperty('--my')
    })
    card.addEventListener('click', () => {
      document.querySelectorAll('.mh-icard.is-focused').forEach((activeCard) => {
        if (activeCard !== card) activeCard.classList.remove('is-focused')
      })
      card.classList.toggle('is-focused')
    })
  })

  // Navbar compact on scroll
  const nav = document.getElementById('mhNav')
  if (nav) {
    window.addEventListener(
      'scroll',
      () => {
        nav.classList.toggle('mh-nav--scrolled', window.scrollY > 80)
      },
      { passive: true },
    )
  }
})

const navigateTo = (path) => {
  router.push(path)
}
</script>

<template>
  <div>
    <NavBar :show-alert-button="false" />

    <main>
      <!-- ══════════════════════════════════════════
           1. HERO — full bleed bg.jpg + frosted panel
      ══════════════════════════════════════════ -->
      <section class="mh-hero mh-hero--welcome" aria-label="Homepage hero">
        <!-- Background image + gradient overlay -->
        <div class="mh-hero-bg" aria-hidden="true"></div>

        <!-- Orbs sit ABOVE the overlay, below the content -->
        <div class="mh-orb mh-orb--1" aria-hidden="true"></div>
        <div class="mh-orb mh-orb--2" aria-hidden="true"></div>
        <div class="mh-orb mh-orb--3" aria-hidden="true"></div>

        <!-- ── Floating ambient badges — continuous up/down animation ── -->
        <div class="mh-fb mh-fb--1" aria-hidden="true">
          <span class="mh-fb-icon">🗺️</span>
          <div class="mh-fb-text">
            <strong>Heat Map</strong>
            <span>Compare suburb signals</span>
          </div>
        </div>
        <div class="mh-fb mh-fb--2" aria-hidden="true">
          <span class="mh-fb-icon">🏛️</span>
          <div class="mh-fb-text">
            <strong>Cool Refuge</strong>
            <span>Find indoor places</span>
          </div>
        </div>
        <div class="mh-fb mh-fb--3" aria-hidden="true">
          <span class="mh-fb-icon">🧭</span>
          <div class="mh-fb-text">
            <strong>Trip Coach</strong>
            <span>Plan a better route</span>
          </div>
        </div>
        <div class="mh-fb mh-fb--4" aria-hidden="true">
          <span class="mh-fb-icon">👕</span>
          <div class="mh-fb-text">
            <strong>Clothing</strong>
            <span>Choose safer layers</span>
          </div>
        </div>
        <div class="mh-fb mh-fb--5" aria-hidden="true">
          <span class="mh-fb-icon">📝</span>
          <div class="mh-fb-text">
            <strong>Safety Plan</strong>
            <span>Prepare before going out</span>
          </div>
        </div>

        <div class="mh-hero-inner">
          <!-- Left: copy -->
          <div class="mh-hero-copy mh-welcome-copy">
            <p class="mh-label">For older adults, families, and carers</p>
            <h1>Feel ready for<br /><em>hot Melbourne days.</em></h1>
            <p class="mh-hero-desc">
              MelbCool is a calm guide for checking heat, finding a cool place, and making a simple
              plan before the day gets too hot.
            </p>
            <div class="mh-hero-ctas">
              <a class="mh-btn primary" @click.prevent="navigateTo('/safety-plan')" href="#"
                >Create My Safety Plan</a
              >
              <a class="mh-btn outline" @click.prevent="navigateTo('/cool-refuges')" href="#"
                >Find a Cool Place</a
              >
            </div>
            <!-- <div class="mh-welcome-steps" aria-label="How MelbCool helps">
              <div>
                <strong>1</strong>
                <span>Check today's heat</span>
              </div>
              <div>
                <strong>2</strong>
                <span>Choose shade or a cool refuge</span>
              </div>
              <div>
                <strong>3</strong>
                <span>Share a simple plan</span>
              </div>
            </div> -->
            <div class="mh-hero-chips">
              <div class="mh-chip">
                <strong>5</strong>
                <span>connected tools for hot days</span>
              </div>
              <div class="mh-chip">
                <strong>12</strong>
                <span>simple safety-plan questions</span>
              </div>
            </div>
          </div>

          <!-- Right: friendly support card -->
          <div
            class="mh-hero-panel mh-welcome-card"
            aria-label="Welcoming MelbCool support preview"
          >
            <div class="mh-welcome-photo">
              <img
                src="/assets/generated-home/hero-welcome.png"
                alt="Older adults and a carer preparing calmly for a hot Melbourne day"
              />
            </div>
            <div class="mh-care-note">
              <span>Today's gentle reminder</span>
              <strong>Plan before the hottest part of the day.</strong>
              <ul>
                <li>Drink water early.</li>
                <li>Call or message someone.</li>
                <li>Keep a cool indoor option ready.</li>
              </ul>
            </div>
          </div>
        </div>
        <div class="mh-hero-scroll" aria-hidden="true"><span></span></div>
      </section>

      <!-- ══════════════════════════════════════════
           FEATURE RIBBON — continuously scrolling
      ══════════════════════════════════════════ -->
      <div class="mh-ticker" aria-hidden="true">
        <div class="mh-ticker-track">
          <!-- duplicated for seamless loop -->
          <span class="mh-tick mh-tick--high">🗺️ Heat Map · suburb overview</span>
          <span class="mh-tick-sep">·</span>
          <span class="mh-tick mh-tick--mod">🏛️ Cool Refuge · indoor places</span>
          <span class="mh-tick-sep">·</span>
          <span class="mh-tick mh-tick--mod">🧭 Trip Coach · route guidance</span>
          <span class="mh-tick-sep">·</span>
          <span class="mh-tick mh-tick--low">👕 Clothing Advisor · outfit choices</span>
          <span class="mh-tick-sep">·</span>
          <span class="mh-tick mh-tick--safe">📝 Safety Plan · simple questions</span>
          <span class="mh-tick-sep">·</span>
          <span class="mh-tick mh-tick--high">📞 Check-in · share your plan</span>
          <span class="mh-tick-sep">·</span>
          <span class="mh-tick mh-tick--mod">💧 Reminders · drink and rest</span>
          <span class="mh-tick-sep">·</span>
          <span class="mh-tick mh-tick--safe">🌳 Shade · avoid exposed streets</span>
          <span class="mh-tick-sep">·</span>
          <span class="mh-tick mh-tick--high">🧊 Cooling · prepare early</span>
          <span class="mh-tick-sep">·</span>
          <span class="mh-tick mh-tick--mod">👥 Carers · support older adults</span>
          <span class="mh-tick-sep">·</span>
          <!-- duplicate for seamless wrap -->
          <span class="mh-tick mh-tick--high">🗺️ Heat Map · suburb overview</span>
          <span class="mh-tick-sep">·</span>
          <span class="mh-tick mh-tick--mod">🏛️ Cool Refuge · indoor places</span>
          <span class="mh-tick-sep">·</span>
          <span class="mh-tick mh-tick--mod">🧭 Trip Coach · route guidance</span>
          <span class="mh-tick-sep">·</span>
          <span class="mh-tick mh-tick--low">👕 Clothing Advisor · outfit choices</span>
          <span class="mh-tick-sep">·</span>
          <span class="mh-tick mh-tick--safe">📝 Safety Plan · simple questions</span>
          <span class="mh-tick-sep">·</span>
          <span class="mh-tick mh-tick--high">📞 Check-in · share your plan</span>
          <span class="mh-tick-sep">·</span>
          <span class="mh-tick mh-tick--mod">💧 Reminders · drink and rest</span>
          <span class="mh-tick-sep">·</span>
          <span class="mh-tick mh-tick--safe">🌳 Shade · avoid exposed streets</span>
          <span class="mh-tick-sep">·</span>
          <span class="mh-tick mh-tick--high">🧊 Cooling · prepare early</span>
          <span class="mh-tick-sep">·</span>
          <span class="mh-tick mh-tick--mod">👥 Carers · support older adults</span>
          <span class="mh-tick-sep">·</span>
        </div>
      </div>

      <!-- ══════════════════════════════════════════
           2. IMPACT NUMBERS — stat cards with visualizations
      ══════════════════════════════════════════ -->
      <section class="mh-impact" aria-label="Key heat statistics">
        <div class="mh-impact-head" data-animate>
          <p class="mh-label">By the numbers</p>
          <h2>Why heat hits older adults <em>harder</em></h2>
        </div>
        <div class="mh-impact-grid mh-stagger" data-animate>
          <div
            class="mh-icard mh-icard--red mh-icard--age"
            tabindex="0"
            aria-label="80 percent of heat hospital stays involve adults over 65"
            @click="toggleCard(0)"
          >
            <span class="mh-icard-kicker">Hospital stays</span>
            <strong class="mh-icard-num"><span class="mh-count" data-target="80">0</span>%</strong>
            <p class="mh-icard-label">of heat hospital stays involve adults over 65</p>
            <div v-show="expandedCards[0]" class="mh-data-visual mh-age-orbit" aria-hidden="true">
              <div class="mh-age-ring"><span>8 of 10</span></div>
              <div class="mh-age-dots">
                <i></i><i></i><i></i><i></i><i></i><i></i><i></i><i></i><b></b><b></b>
              </div>
            </div>
            <span v-show="expandedCards[0]" class="mh-viz-note">Hover or tap to spotlight the older-adult share</span>
            <span class="mh-expand-icon">{{ expandedCards[0] ? '▲' : '▼' }}</span>
          </div>

          <div
            class="mh-icard mh-icard--orange mh-icard--street"
            tabindex="0"
            aria-label="Tree shade changes how comfortable a street feels"
            @click="toggleCard(1)"
          >
            <span class="mh-icard-kicker">Street heat</span>
            <strong class="mh-icard-num">Shade</strong>
            <p class="mh-icard-label">tree canopy can make outdoor movement feel safer</p>
            <div v-show="expandedCards[1]" class="mh-data-visual mh-street-scene" aria-hidden="true">
              <div class="mh-park-zone">
                <span>Shaded<br />rest</span><i></i>
              </div>
              <div class="mh-heat-wave"><span></span><span></span><span></span></div>
              <div class="mh-street-zone">
                <span>Exposed<br />street</span><i></i><i></i><i></i>
              </div>
            </div>
            <span v-show="expandedCards[1]" class="mh-viz-note">Move across the card to compare shade and exposure</span>
            <span class="mh-expand-icon">{{ expandedCards[1] ? '▲' : '▼' }}</span>
          </div>

          <div
            class="mh-icard mh-icard--blue mh-icard--water"
            tabindex="0"
            aria-label="Hydration loss can be twice as fast for seniors during heatwaves"
            @click="toggleCard(2)"
          >
            <span class="mh-icard-kicker">Hydration</span>
            <strong class="mh-icard-num"><span class="mh-count" data-target="2">0</span>×</strong>
            <p class="mh-icard-label">faster hydration loss for seniors during heatwaves</p>
            <div v-show="expandedCards[2]" class="mh-data-visual mh-water-visual" aria-hidden="true">
              <div class="mh-water-glass mh-water-glass--normal"><span></span><em>Others</em></div>
              <div class="mh-water-flow"><i></i><i></i><i></i></div>
              <div class="mh-water-glass mh-water-glass--senior"><span></span><em>Seniors</em></div>
            </div>
            <span v-show="expandedCards[2]" class="mh-viz-note">Animated water level shows why reminders matter</span>
            <span class="mh-expand-icon">{{ expandedCards[2] ? '▲' : '▼' }}</span>
          </div>

          <div
            class="mh-icard mh-icard--teal mh-icard--cooling"
            tabindex="0"
            aria-label="Natural cooling ability can reduce by 30 percent with age"
            @click="toggleCard(3)"
          >
            <span class="mh-icard-kicker">Cooling ability</span>
            <strong class="mh-icard-num"><span class="mh-count" data-target="30">0</span>%</strong>
            <p class="mh-icard-label">
              reduction in the body's natural ability to cool down with age
            </p>
            <div v-show="expandedCards[3]" class="mh-data-visual mh-cooling-dial" aria-hidden="true">
              <div class="mh-dial">
                <span></span>
                <b>70%</b>
              </div>
              <div class="mh-cooling-beads"><i></i><i></i><i></i><i></i><i></i></div>
            </div>
            <span v-show="expandedCards[3]" class="mh-viz-note"
              >The dial compares younger cooling capacity with senior capacity</span
            >
            <span class="mh-expand-icon">{{ expandedCards[3] ? '▲' : '▼' }}</span>
          </div>
        </div>
      </section>

      <!-- ══════════════════════════════════════════
           3. SUBURB RISK COMPARISON
      ══════════════════════════════════════════ -->
      <section class="mh-suburbs" aria-label="Suburb heat risk">
        <div class="mh-suburbs-inner">
          <div class="mh-two-col">
            <div class="mh-section-copy" data-animate="from-left">
              <p class="mh-label">How the tools connect</p>
              <h2>Choose the feature<br /><em>that helps next.</em></h2>
              <p>
                The homepage explains the journey. Open the feature pages when you want actual
                suburb readings, weather conditions, route timing, or a personal plan.
              </p>
              <div class="mh-copy-stats">
                <div class="mh-copy-stat"><strong>60+</strong><span>suburbs tracked</span></div>
                <div class="mh-copy-stat"><strong>5</strong><span>linked safety tools</span></div>
              </div>
              <a
                class="mh-btn primary"
                @click.prevent="navigateTo('/heatmap')"
                href="#"
                style="margin-top: 1.75rem"
                >Open Full Heat Map →</a
              >
            </div>

            <div
              class="mh-tool-orbit"
              data-animate="from-right"
              aria-label="MelbCool feature flow preview"
            >
              <div class="mh-tool-rings" aria-hidden="true">
                <span></span><span></span><span></span>
              </div>
              <div class="mh-tool-core">
                <span>MelbCool</span>
                <strong>Choose next step</strong>
              </div>
              <a
                class="mh-tool-node mh-tool-node--map"
                @click.prevent="navigateTo('/heatmap')"
                href="#"
              >
                <b>01</b><strong>Heat Map</strong><span>Suburb overview</span>
              </a>
              <a
                class="mh-tool-node mh-tool-node--refuge"
                @click.prevent="navigateTo('/cool-refuges')"
                href="#"
              >
                <b>02</b><strong>Cool Refuge</strong><span>Nearby places</span>
              </a>
              <a
                class="mh-tool-node mh-tool-node--trip"
                @click.prevent="navigateTo('/trip-coach')"
                href="#"
              >
                <b>03</b><strong>Trip Coach</strong><span>Safer movement</span>
              </a>
              <a
                class="mh-tool-node mh-tool-node--clothes"
                @click.prevent="navigateTo('/outfit-advisor')"
                href="#"
              >
                <b>04</b><strong>Clothing Advisor</strong><span>Outfit support</span>
              </a>
              <a
                class="mh-tool-node mh-tool-node--plan"
                @click.prevent="navigateTo('/safety-plan')"
                href="#"
              >
                <b>05</b><strong>Safety Plan</strong><span>Personal steps</span>
              </a>
              <div class="mh-tool-note">
                Feature previews only. Open each tool for real suburb, weather, and personal
                recommendations.
              </div>
            </div>
          </div>
        </div>
      </section>

      <!-- ══════════════════════════════════════════
           4. TRIP TIMING PREVIEW
      ══════════════════════════════════════════ -->
      <!-- <section class="mh-timing" aria-label="Trip timing preview">
        <div class="mh-timing-inner">
          <div class="mh-timing-head" data-animate>
            <div>
              <p class="mh-label light">Time your trips right</p>
              <h2>Preview safer<br /><em>planning windows.</em></h2>
              <p>This homepage shows the idea only. Trip Coach calculates the actual timing for the selected suburb, route, and transport mode.</p>
            </div>
            <a class="mh-btn outline" @click.prevent="navigateTo('/trip-coach')" href="#">Plan My Trip →</a>
          </div>

          <div class="mh-window-map" data-animate aria-label="Conceptual trip planning window preview">
            <div class="mh-window-ribbon" aria-hidden="true"></div>
            <article class="mh-window-card mh-window-card--easy">
              <b>1</b>
              <strong>Start early</strong>
              <span>Prepare, leave slowly, and keep the trip simple.</span>
            </article>
            <article class="mh-window-card mh-window-card--care">
              <b>2</b>
              <strong>Check first</strong>
              <span>Look for shade, transport wait time, and rest stops.</span>
            </article>
            <article class="mh-window-card mh-window-card--avoid">
              <b>3</b>
              <strong>Pause the trip</strong>
              <span>If the day feels harsh, choose indoor tasks instead.</span>
            </article>
            <article class="mh-window-card mh-window-card--better">
              <b>4</b>
              <strong>Try later</strong>
              <span>Re-check the route when conditions start easing.</span>
            </article>
            <div class="mh-window-key">
              <span class="good">Easier window</span>
              <span class="watch">Check first</span>
              <span class="delay">Delay if needed</span>
            </div>
          </div>
        </div>
      </section> -->

      <section class="mh-timing" aria-label="Hourly heat pattern">
        <div class="mh-timing-inner">
          <div class="mh-timing-head" data-animate>
            <div>
              <p class="mh-label light">Time your trips right</p>
              <h2>When is it<br /><em>safe to go out?</em></h2>
              <p>
                Melbourne heat peaks between 10am and 4pm. Trip Coach tells you exactly when
                conditions are safer for your specific suburb and route.
              </p>
            </div>
            <a class="mh-btn outline" href="/trip-coach">Plan My Trip →</a>
          </div>

          <div class="mh-hourly mh-hourly-chart" data-animate>
            <div class="mh-best-window" aria-label="Best window to go out">
              ✓ Best window to go out
            </div>
            <!-- Animated sun cursor sweeps across the chart -->
            <div class="mh-sun-cursor" aria-hidden="true">
              <i class="fa-solid fa-sun mh-sun-icon"></i>
              <div class="mh-sun-line"></div>
            </div>
            <div
              class="mh-hourly-bars"
              role="img"
              aria-label="Example heat pattern on a hot Melbourne day"
            >
              <div class="mh-hbar-group" style="--col: #22c55e; --h: 26%; --d: 0s">
                <span class="mh-hbar-temp">22°</span>
                <div class="mh-hbar-fill"></div>
                <span class="mh-hbar-hour">6am</span>
              </div>
              <div class="mh-hbar-group" style="--col: #4ade80; --h: 34%; --d: 0.1s">
                <span class="mh-hbar-temp">25°</span>
                <div class="mh-hbar-fill"></div>
                <span class="mh-hbar-hour">8am</span>
              </div>
              <div class="mh-hbar-group" style="--col: #fbbf24; --h: 50%; --d: 0.2s">
                <span class="mh-hbar-temp">30°</span>
                <div class="mh-hbar-fill"></div>
                <span class="mh-hbar-hour">10am</span>
              </div>
              <div class="mh-hbar-group" style="--col: #f97316; --h: 65%; --d: 0.3s">
                <span class="mh-hbar-temp">34°</span>
                <div class="mh-hbar-fill"></div>
                <span class="mh-hbar-hour">12pm</span>
              </div>
              <div class="mh-hbar-group" style="--col: #ef4444; --h: 90%; --d: 0.4s">
                <span class="mh-hbar-temp">38°</span>
                <div class="mh-hbar-fill"></div>
                <span class="mh-hbar-hour">2pm</span>
              </div>
              <div class="mh-hbar-group" style="--col: #ef4444; --h: 82%; --d: 0.5s">
                <span class="mh-hbar-temp">37°</span>
                <div class="mh-hbar-fill"></div>
                <span class="mh-hbar-hour">4pm</span>
              </div>
              <div class="mh-hbar-group" style="--col: #f97316; --h: 56%; --d: 0.6s">
                <span class="mh-hbar-temp">32°</span>
                <div class="mh-hbar-fill"></div>
                <span class="mh-hbar-hour">6pm</span>
              </div>
              <div class="mh-hbar-group" style="--col: #84cc16; --h: 38%; --d: 0.7s">
                <span class="mh-hbar-temp">27°</span>
                <div class="mh-hbar-fill"></div>
                <span class="mh-hbar-hour">8pm</span>
              </div>
            </div>
            <div class="mh-timing-legend">
              <span style="color: #4ade80">● Safe — before 9am or after 7pm</span>
              <span style="color: #fbbf24">● Caution — 9am–11am</span>
              <span style="color: #ef4444">● Danger — 11am–6pm peak heat</span>
            </div>
          </div>
        </div>
      </section>

      <!-- ══════════════════════════════════════════
           5. FEATURE SECTIONS — full-width alternating
      ══════════════════════════════════════════ -->

      <!-- 5A — Heat Map -->
      <section class="mh-feature" id="feat-heatmap" aria-label="Heat Map tool">
        <div class="mh-feature-inner">
          <div class="mh-feature-visual" data-animate="from-left">
            <img src="/assets/original/reli2.png" alt="Melbourne suburb heat map" loading="lazy" />
            <div class="mh-feature-img-tag"><span class="mh-pulse-dot"></span> Feature preview</div>
          </div>
          <div class="mh-feature-copy" data-animate="from-right">
            <span class="mh-feature-num">01</span>
            <p class="mh-label">Heat Map</p>
            <h2>Explore suburb heat patterns clearly.</h2>
            <p>
              The Heat Map page is where users open the real suburb view. The homepage only explains
              what the map helps them compare.
            </p>
            <div class="mh-simple-risk" aria-label="Heat Map feature preview">
              <div class="mh-risk-row">
                <span class="mh-risk-dot" style="background: #0d9488"></span>
                <span class="mh-risk-name">Suburb comparison</span>
                <span class="mh-risk-level" style="color: #0f766e; background: #ecfdf5">Map</span>
              </div>
              <div class="mh-risk-row">
                <span class="mh-risk-dot" style="background: #65a30d"></span>
                <span class="mh-risk-name">Tree canopy context</span>
                <span class="mh-risk-level" style="color: #65a30d; background: #f7fee7">Shade</span>
              </div>
              <div class="mh-risk-row">
                <span class="mh-risk-dot" style="background: #2563eb"></span>
                <span class="mh-risk-name">Older-adult support</span>
                <span class="mh-risk-level" style="color: #2563eb; background: #eff6ff">Care</span>
              </div>
            </div>
            <a class="mh-btn primary" @click.prevent="navigateTo('/heatmap')" href="#"
              >Open Heat Map →</a
            >
          </div>
        </div>
      </section>

      <!-- 5B — Cool Refuges (alt bg, flipped) -->
      <section
        class="mh-feature mh-feature--alt mh-feature--flip"
        id="feat-refuges"
        aria-label="Cool Refuges tool"
      >
        <div class="mh-feature-inner">
          <div class="mh-feature-visual" data-animate="from-right">
            <img
              src="/assets/generated-home/cool-refuge.png"
              alt="Welcoming cool indoor refuge for older adults"
              loading="lazy"
            />
            <div class="mh-feature-img-tag" style="background: rgba(14, 116, 144, 0.9)">
              ❄️ Air conditioned
            </div>
          </div>
          <div class="mh-feature-copy" data-animate="from-left">
            <span class="mh-feature-num">02</span>
            <p class="mh-label">Cool Refuge</p>
            <h2>Find your nearest cool space in seconds.</h2>
            <p>
              Libraries, community hubs, shopping centres, parks, and museums — every type of public
              cooling space, mapped, filtered, and ranked by how close and how accessible they are.
            </p>
            <div class="mh-refuge-list" aria-label="Example cool refuges">
              <div class="mh-refuge-row">
                <img src="/assets/generated-home/cool-refuge.png" alt="" />
                <div>
                  <strong>State Library Victoria</strong>
                  <span>Library seating · water access · free entry</span>
                </div>
              </div>
              <div class="mh-refuge-row">
                <img src="/assets/generated-home/personal-needs.png" alt="" />
                <div>
                  <strong>Neighbourhood Community Hub</strong>
                  <span>Community support · toilets · seating</span>
                </div>
              </div>
              <div class="mh-refuge-row">
                <img src="/assets/generated-home/shade-signal.png" alt="" />
                <div>
                  <strong>Indoor Shopping Centre</strong>
                  <span>Indoor walking · rest areas · food nearby</span>
                </div>
              </div>
            </div>
            <a class="mh-btn primary" @click.prevent="navigateTo('/cool-refuges')" href="#"
              >Find Cool Refuge →</a
            >
          </div>
        </div>
      </section>

      <!-- 5C — Trip Coach -->
      <section class="mh-feature" id="feat-trip" aria-label="Trip Coach tool">
        <div class="mh-feature-inner">
          <div class="mh-feature-visual" data-animate="from-left">
            <img
              src="/assets/generated-home/trip-coach.png"
              alt="Older adult and carer planning a shaded route"
              loading="lazy"
            />
          </div>
          <div class="mh-feature-copy" data-animate="from-right">
            <span class="mh-feature-num">03</span>
            <p class="mh-label">Trip Coach</p>
            <h2>Leave at the right moment, every time.</h2>
            <p>
              Your trip is broken into legs — walk, wait, ride, arrive. Each leg gets an exposure
              score based on shade, transport type, and the time of day. Trip Coach tells you when
              and how to travel with the least heat exposure.
            </p>
            <div class="mh-time-blocks" aria-label="Safe and unsafe travel windows">
              <div class="mh-time-block mh-time-block--safe">
                <span class="mh-time-icon">🌅</span>
                <div>
                  <strong>Before 9am</strong>
                  <span>Start early and reduce outdoor exposure</span>
                </div>
              </div>
              <div class="mh-time-block mh-time-block--danger">
                <span class="mh-time-icon">☀️</span>
                <div>
                  <strong>10am – 4pm</strong>
                  <span>Consider delaying if conditions are harsh</span>
                </div>
              </div>
              <div class="mh-time-block mh-time-block--safe">
                <span class="mh-time-icon">🌆</span>
                <div>
                  <strong>After 6pm</strong>
                  <span>Check the tool before leaving</span>
                </div>
              </div>
            </div>
            <a class="mh-btn primary" @click.prevent="navigateTo('/trip-coach')" href="#"
              >Plan My Trip →</a
            >
          </div>
        </div>
      </section>

      <!-- 5D — Clothing Advisor (alt, flipped) -->
      <section
        class="mh-feature mh-feature--alt mh-feature--flip"
        id="feat-clothing"
        aria-label="Clothing Advisor tool"
      >
        <div class="mh-feature-inner">
          <div class="mh-feature-visual" data-animate="from-right">
            <img
              src="/assets/generated-home/clothing-advisor.png"
              alt="Older adult choosing lightweight heat-safe clothing"
              loading="lazy"
            />
          </div>
          <div class="mh-feature-copy" data-animate="from-left">
            <span class="mh-feature-num">04</span>
            <p class="mh-label">Clothing Advisor</p>
            <h2>Dressed right for the day you are planning.</h2>
            <p>
              The Clothing Advisor page uses the actual conditions. The homepage only shows the kind
              of choices it helps older adults make.
            </p>
            <div class="mh-clothing-check" aria-label="Clothing checklist preview">
              <p class="mh-clothing-header">Preview · clothing choices</p>
              <div class="mh-check-row">
                <span class="mh-check-yes">✓</span>
                <span>Lightweight, light-coloured fabrics</span>
              </div>
              <div class="mh-check-row">
                <span class="mh-check-yes">✓</span>
                <span>UPF 50+ sun shirt or loose long sleeves</span>
              </div>
              <div class="mh-check-row">
                <span class="mh-check-yes">✓</span>
                <span>Wide-brimmed hat (min. 7.5cm brim)</span>
              </div>
              <div class="mh-check-row">
                <span class="mh-check-no">✗</span>
                <span>Dark, synthetic, or tight-fitting layers</span>
              </div>
            </div>
            <a class="mh-btn primary" @click.prevent="navigateTo('/outfit-advisor')" href="#"
              >Get Recommendations →</a
            >
          </div>
        </div>
      </section>

      <!-- 5E — Safety Plan (dark full-width) -->
      <section class="mh-feature mh-feature--dark" id="feat-plan" aria-label="Safety Plan tool">
        <div class="mh-feature-inner">
          <div class="mh-feature-visual" data-animate="from-left">
            <img
              src="/assets/generated-home/safety-plan.png"
              alt="Older adult and carer reviewing a personal heat safety plan"
              loading="lazy"
            />
            <div
              class="mh-feature-img-tag"
              style="
                background: rgba(163, 247, 125, 0.18);
                color: #a3f77d;
                border: 1px solid rgba(163, 247, 125, 0.3);
              "
            >
              📋 Printable plan
            </div>
          </div>
          <div class="mh-feature-copy mh-feature-copy--light" data-animate="from-right">
            <span class="mh-feature-num" style="color: rgba(163, 247, 125, 0.6)">05</span>
            <p class="mh-label">Personal Safety Plan</p>
            <h2>6 questions.<br />One personalised plan.</h2>
            <p>
              Answer 6 simple questions about your suburb, health, and daily routine. Get a calm,
              clear plan with your risk level, safest time window, nearest cool refuge, and
              emergency contacts — all printable.
            </p>
            <div class="mh-plan-flow" aria-label="Plan creation steps">
              <div class="mh-plan-step">
                <span class="mh-plan-step-num">1</span>
                <span>Your suburb & health</span>
              </div>
              <div class="mh-plan-arr">→</div>
              <div class="mh-plan-step">
                <span class="mh-plan-step-num">2</span>
                <span>Risk level</span>
              </div>
              <div class="mh-plan-arr">→</div>
              <div class="mh-plan-step">
                <span class="mh-plan-step-num">3</span>
                <span>Your plan</span>
              </div>
            </div>
            <a class="mh-btn primary" @click.prevent="navigateTo('/safety-plan')" href="#"
              >Create My Plan →</a
            >
          </div>
        </div>
      </section>

      <!-- ══════════════════════════════════════════
           6. TRANSPORT RISK
      ══════════════════════════════════════════ -->
      <!-- <section class="mh-transport" aria-label="Transport heat exposure">
        <div class="mh-transport-inner">
          <div class="mh-section-center" data-animate>
            <p class="mh-label light">Getting around on hot days</p>
            <h2>How you travel<br /><em>changes your risk.</em></h2>
            <p>The same trip can feel very different depending on shade, waiting time, and transport mode. The detailed route calculation belongs in Trip Coach.</p>
          </div>
          <div class="mh-transport-grid mh-stagger">
            <div class="mh-tcard">
              <div class="mh-tcard-top" style="background:rgba(239,68,68,0.12)">
                <div class="mh-tcard-mark mh-tcard-mark--walk" aria-label="Walking">
                  <svg viewBox="0 0 48 48" aria-hidden="true">
                    <path d="M25 10c2.8 0 5 2.2 5 5s-2.2 5-5 5-5-2.2-5-5 2.2-5 5-5Z" />
                    <path d="M22 23 17 34l-6 4" />
                    <path d="m25 23 6 7 6 2" />
                    <path d="m21 28 8 10" />
                  </svg>
                </div>
                <span class="mh-tbadge" style="color:#dc2626;background:rgba(220,38,38,0.12)">High Risk</span>
              </div>
              <h3>Walking</h3>
              <p>Direct sun and no shelter can make this the hardest option. Use Trip Coach before choosing a walking route.</p>
              <div class="mh-route-viz mh-route-viz--high" aria-hidden="true">
                <div class="mh-route-path">
                  <span></span><span></span><span></span><span></span>
                </div>
                <div class="mh-route-tags">
                  <span>Open sun</span><span>No rest</span><span>Long walk</span>
                </div>
              </div>
              <span class="mh-tcard-score">Highest exposure pattern</span>
            </div>
            <div class="mh-tcard mh-tcard--featured">
              <div class="mh-tcard-top" style="background:rgba(234,88,12,0.12)">
                <div class="mh-tcard-mark mh-tcard-mark--stop" aria-label="Public transport stop">
                  <svg viewBox="0 0 48 48" aria-hidden="true">
                    <path d="M15 14h18a5 5 0 0 1 5 5v12a5 5 0 0 1-5 5H15a5 5 0 0 1-5-5V19a5 5 0 0 1 5-5Z" />
                    <path d="M16 22h16" />
                    <path d="M17 36v4" />
                    <path d="M31 36v4" />
                    <path d="M17 29h.2" />
                    <path d="M31 29h.2" />
                    <path d="M20 9h8" />
                  </svg>
                </div>
                <span class="mh-tbadge" style="color:#c2410c;background:rgba(194,65,12,0.12)">Moderate</span>
              </div>
              <h3>Tram / Bus</h3>
              <p>Indoor travel can help, but waiting outside still matters. Plan around a shaded stop and shorter transfer.</p>
              <div class="mh-route-viz mh-route-viz--medium" aria-hidden="true">
                <div class="mh-route-path">
                  <span></span><span></span><span></span><span></span>
                </div>
                <div class="mh-route-tags">
                  <span>Stop wait</span><span>Short shade</span><span>Transfer</span>
                </div>
              </div>
              <span class="mh-tcard-score">Medium exposure pattern</span>
            </div>
            <div class="mh-tcard">
              <div class="mh-tcard-top" style="background:rgba(34,197,94,0.12)">
                <div class="mh-tcard-mark mh-tcard-mark--door" aria-label="Door-to-door travel">
                  <svg viewBox="0 0 48 48" aria-hidden="true">
                    <path d="M14 40V10h17l5 4v26" />
                    <path d="M20 40V16h16" />
                    <path d="M30 27h.2" />
                    <path d="M10 40h30" />
                  </svg>
                </div>
                <span class="mh-tbadge" style="color:#16a34a;background:rgba(22,163,74,0.12)">Low Risk</span>
              </div>
              <h3>Driving</h3>
              <p>Door-to-door travel can reduce outdoor exposure. Park in shade and make the trip easier for older passengers.</p>
              <div class="mh-route-viz mh-route-viz--low" aria-hidden="true">
                <div class="mh-route-path">
                  <span></span><span></span><span></span><span></span>
                </div>
                <div class="mh-route-tags">
                  <span>Door</span><span>Shade</span><span>Rest ready</span>
                </div>
              </div>
              <span class="mh-tcard-score">Lower exposure pattern</span>
            </div>
          </div>
          <p class="mh-transport-note">
            Trip Coach calculates exposure for your specific route, time, and transport mode. &nbsp;<a @click.prevent="navigateTo('/trip-coach')" href="#">Plan my trip →</a>
          </p>
        </div>
      </section> -->

      <!-- ══════════════════════════════════════════
           7. RISK FACTORS — editorial horizontal cards
      ══════════════════════════════════════════ -->
      <!-- <section class="mh-risk" aria-label="How heat risk is calculated">
        <div class="mh-risk-inner">
          <div class="mh-risk-lead" data-animate>
            <p class="mh-label">Evidence-based</p>
            <h2>Three signals.<br />One clearer decision.</h2>
            <p>This homepage explains the inputs at a high level. The feature pages show the actual readings and calculations.</p>
          </div>

          <div class="mh-factors">
            <div class="mh-factor" data-animate>
              <div class="mh-factor-img">
                <img src="/assets/generated-home/conditions-signal.png" alt="Melbourne street conditions and shaded resting area" loading="lazy" />
                <div class="mh-factor-num-overlay">01</div>
              </div>
              <div class="mh-factor-body">
                <div class="mh-factor-accent" style="--accent:#ef4444"></div>
                <div class="mh-factor-tag" style="color:#dc2626;background:#fef2f2">Weather signal</div>
                <h3>Current conditions</h3>
                <p>The feature pages use actual weather conditions to help decide when to rest, travel, dress differently, or choose an indoor place.</p>
                <div class="mh-signal-steps" aria-label="Weather signal steps">
                  <span>Check</span><span>Plan</span><span>Act</span><span>Review</span>
                  <p class="mh-factor-caption">Exact readings are shown inside the feature pages</p>
                </div>
              </div>
            </div>

            <div class="mh-factor mh-factor--flip" data-animate>
              <div class="mh-factor-img">
                <img src="/assets/generated-home/shade-signal.png" alt="Older adult resting under tree canopy shade" loading="lazy" />
                <div class="mh-factor-num-overlay">02</div>
              </div>
              <div class="mh-factor-body">
                <div class="mh-factor-accent" style="--accent:#16a34a"></div>
                <div class="mh-factor-tag" style="color:#15803d;background:#f0fdf4">Tree Coverage</div>
                <h3>Canopy cover percentage</h3>
                <p>Tree canopy helps the map explain where a route may feel more exposed and where shade or indoor cooling might be easier to find.</p>
                <div class="mh-canopy-grid" aria-label="Tree canopy coverage preview">
                  <span></span><span></span><span></span><span></span>
                  <span></span><span></span><span></span><span></span>
                  <span></span><span></span><span></span><span></span>
                  <span></span><span></span><span></span><span></span>
                  <p class="mh-factor-caption">Under 15% canopy cover raises risk significantly</p>
                </div>
              </div>
            </div>

            <div class="mh-factor" data-animate>
              <div class="mh-factor-img">
                <img src="/assets/generated-home/personal-needs.png" alt="Carer checking in with an older adult at home" loading="lazy" />
                <div class="mh-factor-num-overlay">03</div>
              </div>
              <div class="mh-factor-body">
                <div class="mh-factor-accent" style="--accent:#7c3aed"></div>
                <div class="mh-factor-tag" style="color:#6d28d9;background:#f5f3ff">Personal needs</div>
                <h3>Older-adult context</h3>
                <p>Health, mobility, living situation, cooling access, and check-ins change what advice is useful for each person.</p>
                <div class="mh-support-orbit" aria-label="Personal support context preview">
                  <strong>Plan</strong>
                  <span class="mh-support-orbit-home">Home</span>
                  <span class="mh-support-orbit-health">Health</span>
                  <span class="mh-support-orbit-mobility">Mobility</span>
                  <span class="mh-support-orbit-support">Support</span>
                  <p class="mh-factor-caption">Personal questions belong in the Safety Plan tool</p>
                </div>
              </div>
            </div>
          </div>
        </div>
      </section> -->

      <!-- ══════════════════════════════════════════
         6. TRANSPORT RISK
    ══════════════════════════════════════════ -->
      <section class="mh-transport" aria-label="Transport heat exposure">
        <div class="mh-transport-inner">
          <div class="mh-section-center" data-animate>
            <p class="mh-label light">Getting around on hot days</p>
            <h2>How you travel<br /><em>changes your risk.</em></h2>
            <p>
              On a 38°C day, the same 20-minute journey carries very different heat exposure
              depending on how you get there.
            </p>
          </div>
          <div class="mh-transport-grid mh-stagger">
            <!-- Walking — heat haze effect -->
            <div class="mh-tcard mh-tcard--walk">
              <div class="mh-heat-haze" aria-hidden="true">
                <span></span><span></span><span></span><span></span><span></span>
              </div>
              <div class="mh-tcard-top" style="background: rgba(239, 68, 68, 0.12)">
                <div class="mh-tcard-icon mh-icard-badge mh-icard-badge--red mh-ticon--bounce">
                  <i class="fa-solid fa-person-walking"></i>
                </div>
                <span class="mh-tbadge" style="color: #dc2626; background: rgba(220, 38, 38, 0.12)"
                  >High Risk</span
                >
              </div>
              <h3>Walking</h3>
              <p>
                Direct sun, no shelter, maximum heat absorption. The highest-exposure option during
                extreme heat — avoid between 10am and 5pm.
              </p>
              <div class="mh-tcard-dots">
                <i class="fa-solid fa-sun mh-tdot mh-tdot--on" style="--tc: #ef4444"></i>
                <i class="fa-solid fa-sun mh-tdot mh-tdot--on" style="--tc: #ef4444"></i>
                <i class="fa-solid fa-sun mh-tdot mh-tdot--on" style="--tc: #ef4444"></i>
                <i class="fa-solid fa-sun mh-tdot mh-tdot--on" style="--tc: #ef4444"></i>
                <i class="fa-solid fa-sun mh-tdot mh-tdot--on" style="--tc: #ef4444"></i>
                <i class="fa-solid fa-sun mh-tdot mh-tdot--on" style="--tc: #ef4444"></i>
                <i class="fa-solid fa-sun mh-tdot mh-tdot--on" style="--tc: #ef4444"></i>
                <i class="fa-solid fa-sun mh-tdot mh-tdot--on" style="--tc: #ef4444"></i>
                <i class="fa-solid fa-sun mh-tdot mh-tdot--off"></i>
                <i class="fa-solid fa-sun mh-tdot mh-tdot--off"></i>
              </div>
              <span class="mh-tcard-score">8 of 10 · High exposure</span>
            </div>
            <div class="mh-tcard mh-tcard--featured">
              <div class="mh-tcard-top" style="background: rgba(234, 88, 12, 0.12)">
                <div class="mh-tcard-icon mh-icard-badge mh-icard-badge--orange mh-ticon--sway">
                  <i class="fa-solid fa-train-subway"></i>
                </div>
                <span class="mh-tbadge" style="color: #c2410c; background: rgba(194, 65, 12, 0.12)"
                  >Moderate</span
                >
              </div>
              <h3>Tram / Bus</h3>
              <p>
                Air-conditioned cabin significantly reduces exposure, but outdoor waiting time adds
                risk. Plan around a shaded stop and travel mid-trip.
              </p>
              <div class="mh-tcard-dots">
                <i class="fa-solid fa-sun mh-tdot mh-tdot--on" style="--tc: #ea580c"></i>
                <i class="fa-solid fa-sun mh-tdot mh-tdot--on" style="--tc: #ea580c"></i>
                <i class="fa-solid fa-sun mh-tdot mh-tdot--on" style="--tc: #ea580c"></i>
                <i class="fa-solid fa-sun mh-tdot mh-tdot--on" style="--tc: #ea580c"></i>
                <i class="fa-solid fa-sun mh-tdot mh-tdot--on" style="--tc: #ea580c"></i>
                <i class="fa-solid fa-sun mh-tdot mh-tdot--off"></i>
                <i class="fa-solid fa-sun mh-tdot mh-tdot--off"></i>
                <i class="fa-solid fa-sun mh-tdot mh-tdot--off"></i>
                <i class="fa-solid fa-sun mh-tdot mh-tdot--off"></i>
                <i class="fa-solid fa-sun mh-tdot mh-tdot--off"></i>
              </div>
              <span class="mh-tcard-score">5 of 10 · Moderate exposure</span>
            </div>
            <div class="mh-tcard">
              <div class="mh-tcard-top" style="background: rgba(34, 197, 94, 0.12)">
                <div class="mh-tcard-icon mh-icard-badge mh-icard-badge--teal mh-ticon--drive">
                  <i class="fa-solid fa-car-side"></i>
                </div>
                <span class="mh-tbadge" style="color: #16a34a; background: rgba(22, 163, 74, 0.12)"
                  >Low Risk</span
                >
              </div>
              <h3>Driving</h3>
              <p>
                Air-conditioned, door-to-door — the lowest-exposure option during extreme heat. Park
                in shade and pre-cool the car before elderly passengers board.
              </p>
              <div class="mh-tcard-dots">
                <i class="fa-solid fa-sun mh-tdot mh-tdot--on" style="--tc: #22c55e"></i>
                <i class="fa-solid fa-sun mh-tdot mh-tdot--on" style="--tc: #22c55e"></i>
                <i class="fa-solid fa-sun mh-tdot mh-tdot--on" style="--tc: #22c55e"></i>
                <i class="fa-solid fa-sun mh-tdot mh-tdot--off"></i>
                <i class="fa-solid fa-sun mh-tdot mh-tdot--off"></i>
                <i class="fa-solid fa-sun mh-tdot mh-tdot--off"></i>
                <i class="fa-solid fa-sun mh-tdot mh-tdot--off"></i>
                <i class="fa-solid fa-sun mh-tdot mh-tdot--off"></i>
                <i class="fa-solid fa-sun mh-tdot mh-tdot--off"></i>
                <i class="fa-solid fa-sun mh-tdot mh-tdot--off"></i>
              </div>
              <span class="mh-tcard-score">3 of 10 · Low exposure</span>
            </div>
          </div>
          <p class="mh-transport-note">
            Trip Coach calculates exposure for your specific route, time, and transport mode.
            &nbsp; <RouterLink to="/trip-coach">Plan my trip →</RouterLink>
          </p>
        </div>
      </section>

      <!-- ══════════════════════════════════════════
         7. RISK FACTORS — editorial horizontal cards
    ══════════════════════════════════════════ -->
      <section class="mh-risk" aria-label="How heat risk is calculated">
        <div class="mh-risk-inner">
          <div class="mh-risk-lead" data-animate>
            <p class="mh-label">Evidence-based</p>
            <h2>Three factors.<br />One clear risk score.</h2>
            <p>
              Every suburb's risk score is calculated from three real-world data sources updated
              throughout the day.
            </p>
          </div>

          <div class="mh-factors">
            <div class="mh-factor" data-animate>
              <div class="mh-factor-img">
                <img
                  src="/assets/original/temperature.png"
                  alt="Temperature gauge"
                  loading="lazy"
                />
                <div class="mh-factor-num-overlay">01</div>
              </div>
              <div class="mh-factor-body">
                <div class="mh-factor-accent" style="--accent: #ef4444"></div>
                <div class="mh-factor-tag" style="color: #dc2626; background: #fef2f2">
                  Temperature
                </div>
                <h3>Apparent temperature</h3>
                <p>
                  Accounts for humidity and wind chill, not just the air reading. Inner Melbourne
                  regularly hits 36–40°C apparent on extreme summer days — significantly more
                  dangerous than the official maximum.
                </p>
                <div class="mh-factor-bar-wrap">
                  <div class="mh-factor-bar-labels">
                    <span>0°C</span><span>20°C</span><span>30°C</span><span>40°C+</span>
                  </div>
                  <div class="mh-factor-track">
                    <div class="mh-factor-fill" style="--pct: 88%; --col: #ef4444"></div>
                  </div>
                  <p class="mh-factor-caption">36–40°C is the peak risk zone for older adults</p>
                </div>
              </div>
            </div>

            <div class="mh-factor mh-factor--flip" data-animate>
              <div class="mh-factor-img">
                <img
                  src="/assets/original/tree.jpg"
                  alt="Tree canopy providing shade"
                  loading="lazy"
                />
                <div class="mh-factor-num-overlay">02</div>
              </div>
              <div class="mh-factor-body">
                <div class="mh-factor-accent" style="--accent: #16a34a"></div>
                <div class="mh-factor-tag" style="color: #15803d; background: #f0fdf4">
                  Tree Coverage
                </div>
                <h3>Canopy cover percentage</h3>
                <p>
                  Suburbs with less than 15% tree canopy can feel 5–7°C hotter at street level.
                  Shade is the single most effective passive cooling factor — MelbCool uses
                  satellite-derived canopy maps updated quarterly.
                </p>
                <div class="mh-factor-bar-wrap">
                  <div class="mh-factor-bar-labels">
                    <span>0%</span><span>15%</span><span>30%</span><span>50%+</span>
                  </div>
                  <div class="mh-factor-track">
                    <div class="mh-factor-fill" style="--pct: 62%; --col: #16a34a"></div>
                  </div>
                  <p class="mh-factor-caption">Under 15% canopy cover raises risk significantly</p>
                </div>
              </div>
            </div>

            <div class="mh-factor" data-animate>
              <div class="mh-factor-img">
                <img src="/assets/original/uv.jpg" alt="UV sun rays" loading="lazy" />
                <div class="mh-factor-num-overlay">03</div>
                <div class="mh-uv-spin" aria-hidden="true">
                  <i class="fa-solid fa-sun"></i>
                </div>
              </div>
              <div class="mh-factor-body">
                <div class="mh-factor-accent" style="--accent: #7c3aed"></div>
                <div class="mh-factor-tag" style="color: #6d28d9; background: #f5f3ff">
                  UV Index
                </div>
                <h3>Ultraviolet radiation level</h3>
                <p>
                  UV above 3 accelerates skin damage and compounds heat illness risk. Melbourne UV
                  regularly peaks at 10–12 in summer, even on overcast days. Older skin burns faster
                  and sweats less efficiently.
                </p>
                <div class="mh-factor-bar-wrap">
                  <div class="mh-factor-bar-labels">
                    <span>UV 0</span><span>UV 3</span><span>UV 7</span><span>UV 11+</span>
                  </div>
                  <div class="mh-factor-track">
                    <div class="mh-factor-fill" style="--pct: 76%; --col: #7c3aed"></div>
                  </div>
                  <p class="mh-factor-caption">UV 3+ triggers sun-protection requirements</p>
                </div>
              </div>
            </div>
          </div>
        </div>
      </section>

      <!-- ══════════════════════════════════════════
           8. TIPS — with images
      ══════════════════════════════════════════ -->
      <!-- <section class="mh-tips" aria-label="Heat safety tips">
        <div class="mh-tips-inner">
          <div class="mh-tips-head" data-animate>
            <p class="mh-label">Simple actions</p>
            <h2>What to do<br />on a hot day.</h2>
            <p>These actions can reduce your heat risk significantly — especially for adults over 65.</p>
          </div>
          <div class="mh-tips-grid mh-stagger">
            <div class="mh-tip">
              <div class="mh-tip-img">
                <img src="/assets/generated-home/tip-cool-room.png" alt="Cool indoor room prepared for an older adult" loading="lazy" />
              </div>
              <div class="mh-tip-body">
                <h3>🏠 Stay Indoors</h3>
                <ul>
                  <li>Close curtains before midday</li>
                  <li>Stay inside between 12 – 4 PM</li>
                  <li>Use fans or air conditioning</li>
                  <li>Block direct sunlight on west-facing windows</li>
                </ul>
              </div>
            </div>
            <div class="mh-tip">
              <div class="mh-tip-img">
                <img src="/assets/generated-home/tip-water.png" alt="Water and hydration support for hot days" loading="lazy" />
              </div>
              <div class="mh-tip-body">
                <h3>💧 Drink Water</h3>
                <ul>
                  <li>One glass every hour — even without thirst</li>
                  <li>Avoid alcohol and caffeine</li>
                  <li>Eat light — fruit, salads, cold foods</li>
                  <li>Keep a water bottle visible as a reminder</li>
                </ul>
              </div>
            </div>
            <div class="mh-tip">
              <div class="mh-tip-img">
                <img src="/assets/generated-home/tip-checkin.png" alt="Older adult receiving a supportive check-in call" loading="lazy" />
              </div>
              <div class="mh-tip-body">
                <h3>🌳 Find Cool Places</h3>
                <ul>
                  <li>Visit a library or community hub</li>
                  <li>Use shaded paths when outside</li>
                  <li>Check on neighbours and family</li>
                  <li>Call ahead to confirm access before going</li>
                </ul>
              </div>
            </div>
            <div class="mh-tip mh-tip--urgent">
              <div class="mh-tip-img">
                <img src="/assets/generated-home/tip-warning.png" alt="Carer checking on an older adult for warning signs" loading="lazy" />
              </div>
              <div class="mh-tip-body">
                <h3>📞 Emergency Signs</h3>
                <ul>
                  <li>Confusion or sudden dizziness</li>
                  <li>Very hot, dry skin — no sweating</li>
                  <li>Fainting, collapse, or unconscious</li>
                </ul>
                <a class="mh-tip-cta" href="tel:000">Call 000 immediately</a>
              </div>
            </div>
          </div>
        </div>
      </section> -->

      <!-- ══════════════════════════════════════════
           9. FINAL CTA
      ══════════════════════════════════════════ -->
      <!-- <section class="mh-cta" aria-label="Get started with your safety plan">
        <div class="mh-cta-img-bg" aria-hidden="true">
          <img src="/assets/generated-home/hero-welcome.png" alt="" />
          <div class="mh-cta-img-overlay"></div>
        </div>
        <div class="mh-cta-inner" data-animate>
          <div class="mh-cta-copy">
            <p class="mh-label light">Ready when you are</p>
            <h2>Start with one safe choice today.</h2>
            <p>Takes about 3 minutes. You'll walk away with a personalised plan covering your suburb's risk, safest time windows, nearest cool space, and emergency contacts — all in one printable sheet.</p>
          </div>
          <div class="mh-cta-actions">
            <a class="mh-btn primary large" @click.prevent="navigateTo('/safety-plan')" href="#">Make My Plan</a>
            <a class="mh-btn ghost large" @click.prevent="navigateTo('/heatmap')" href="#">Check My Suburb</a>
          </div>
        </div>
        <div class="mh-emergency-strip">
          <span>Emergency signs: confusion, fainting, very hot dry skin →</span>
          <a href="tel:000">Call 000</a>
          <a href="tel:1300606024">Nurse-on-Call</a>
        </div>
      </section> -->
    </main>

    <Footer />
  </div>
</template>
