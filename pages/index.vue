<script setup>
import { ref, onBeforeUnmount, onMounted, nextTick } from 'vue'
import { useGtagConversion } from '~/composables/useGtagConversion'

useHead({
  title: 'Csontkovács, manuálterápia és oszteopátia Budapest - Időpontkérés',
})

// ====== GOOGLE ADS TRACKING ======
function persistClickIdsFromUrl() {
  if (typeof window === 'undefined') return
  const params = new URLSearchParams(window.location.search)
  ;['gclid', 'wbraid', 'gbraid'].forEach((key) => {
    const v = params.get(key)
    if (v) localStorage.setItem(key, v)
  })
}

// ====== PROCESS STEPS (progress indicator) ======
const processStepRefs = ref([null, null, null, null])
const activeProcessStep = ref(1)
let processObserver

const setupProcessObserver = async () => {
  if (typeof window === 'undefined') return

  await nextTick()

  if (processObserver) processObserver.disconnect()

  const els = processStepRefs.value?.filter(Boolean) || []
  if (!els.length) return

  processObserver = new IntersectionObserver(
    (entries) => {
      const visible = entries
        .filter((e) => e.isIntersecting)
        .sort((a, b) => (b.intersectionRatio || 0) - (a.intersectionRatio || 0))

      const top = visible[0]
      if (!top?.target) return

      const step = Number(top.target.getAttribute('data-step'))
      if (!Number.isNaN(step)) activeProcessStep.value = step
    },
    {
      root: null,
      threshold: [0.25, 0.4, 0.55, 0.7],
      rootMargin: '-20% 0px -55% 0px',
    },
  )

  els.forEach((el) => processObserver.observe(el))
}

function getClickIds() {
  if (typeof window === 'undefined')
    return { gclid: null, wbraid: null, gbraid: null }
  return {
    gclid: localStorage.getItem('gclid'),
    wbraid: localStorage.getItem('wbraid'),
    gbraid: localStorage.getItem('gbraid'),
  }
}

// Reactive variables
const isSubmitting = ref(false)
const submitMessage = ref('')
const contactMethod = ref('form') // 'phone' vagy 'form'
const formData = ref({
  serviceType: '',
  propertyType: 'n/a',
  urgency: '',
  name: '',
  email: '',
  phone: '',
  address: '',
  message: '',
})

// ====== FAQ accordion (csak 1 nyitva) ======
const faqRefs = ref([])

const handleFaqToggle = (activeIndex) => {
  const activeEl = faqRefs.value?.[activeIndex]
  if (!activeEl?.open) return

  faqRefs.value.forEach((el, i) => {
    if (i !== activeIndex && el?.open) el.open = false
  })
}

// Form submission handler
const submitForm = async (event) => {
  event.preventDefault()

  if (isSubmitting.value) return

  isSubmitting.value = true
  submitMessage.value = ''

  try {
    const webhookUrl =
      'https://services.leadconnectorhq.com/hooks/bTVDXHCYtnzhI1hdcsrA/webhook-trigger/f5a6a462-f26b-4672-ba7a-9e81b5a5ade9'

    const { gclid, wbraid, gbraid } = getClickIds()
    const serviceName = getServiceDisplayName(formData.value.serviceType)

    const payload = {
      name: formData.value.name,
      email: formData.value.email,
      phone: formData.value.phone,
      address: formData.value.address,
      service_type: formData.value.serviceType,
      property_type: formData.value.propertyType,
      urgency: formData.value.urgency,
      message: formData.value.message,
      gclid,
      wbraid,
      gbraid,
      source: 'Manuálterápia időpontkérő űrlap',
      form_type: 'manual_therapy_booking',
      submission_date: new Date().toISOString(),
      custom_field_1: serviceName,
      custom_field_2: formData.value.propertyType,
      custom_field_3: formData.value.urgency,
    }

    const response = await fetch(webhookUrl, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify(payload),
    })

    if (response.ok) {
      submitMessage.value =
        '✅ Sikeres időpontkérés! Munkanapokon 24 órán belül felvesszük Önnel a kapcsolatot.'

      const { reportConversion } = useGtagConversion()
      reportConversion()

      formData.value = {
        serviceType: '',
        propertyType: 'n/a',
        urgency: '',
        name: '',
        email: '',
        phone: '',
        address: '',
        message: '',
      }
    } else {
      throw new Error('Hiba történt a küldés során')
    }
  } catch (error) {
    console.error('Form submission error:', error)
    submitMessage.value =
      '❌ Hiba történt. Kérjük próbálja újra, vagy hívjon minket!'
  } finally {
    isSubmitting.value = false
  }
}

const getServiceDisplayName = (serviceValue) => {
  const serviceMap = {
    csontkovacs: 'Csontkovács kezelés (gerinc / ízületek)',
    manualterapia: 'Manuálterápia (izom, fascia, mobilizáció)',
    derek_hat: 'Derék- és hátfájdalom kezelése',
    nyak_vall: 'Nyak- és vállfeszülések kezelése',
    testtartas: 'Testtartási problémák kezelése',
    tmj: 'TMJ / állkapocs és sztomatognatikus rendszer',
    zsigeri: 'Zsigeri (viszcerális) oszteopátia',
    sport: 'Sport / aktív életmód melletti mozgásszervi panaszok',
    egyeb: 'Egyéb panasz / konzultáció',
  }
  return serviceMap[serviceValue] || serviceValue
}

const initPage = () => {
  persistClickIdsFromUrl()

  const header = document.querySelector('header')
  const footer = document.querySelector('footer')
  const navbar = document.querySelector('nav')
  const siteChatWidget = document.querySelector('.lc_text-widget')

  if (header) header.style.display = 'none'
  if (footer) footer.style.display = 'none'
  if (navbar) navbar.style.display = 'none'
  if (siteChatWidget) siteChatWidget.style.display = 'none'

  document.querySelectorAll('header, footer, .lc_text-widget').forEach((el) => {
    el.style.display = 'none'
  })
}

onMounted(() => {
  initPage()
  setupProcessObserver()
})

onBeforeUnmount(() => {
  if (processObserver) processObserver.disconnect()
})
</script>

<template>
  <section>
    <div
      class="about-content about-content--subpage-next-format position-relative no-header-footer-page"
    >
      <div class="subpage-content">
        <!-- Fő üzenet - Hero Banner WITH BACKGROUND IMAGE -->
        <div class="trust-banner trust-banner--with-image">
          <div class="banner-content">
            <h2 class="main-title">
              CSONTKOVÁCS ÉS MANUÁLTERÁPIA PEST MEGYÉBEN
            </h2>
            <p class="banner-subtitle">
              <i class="supage-content__p__i"
                >Személyre szabott kezelés • Teljes testet vizsgáló szemlélet •
                Közérthető, bizalomépítő tájékoztatás</i
              >
            </p>
          </div>
        </div>

        <!-- 3 KIEMELT ÉRVELÉS -->
        <div class="benefits-grid">
          <div class="benefit-card">
            <h3>Tapasztalat + integrált szemlélet</h3>
            <p>
              <strong
                >Állapotfelmérésre épülő, célzott manuális kezelés.</strong
              >
              A mozgásszervi panaszok (derék, hát, nyak, váll, ízületek)
              kezelését funkcionális vizsgálat előzi meg, majd személyre szabott
              technikákkal dolgozunk a fájdalom és a beszűkülések csökkentésén.
            </p>
          </div>
          <div class="benefit-card">
            <h3>Időpontfoglalás egyszerűen</h3>
            <p>
              <strong>Közvetlen egyeztetés telefonon vagy űrlapon.</strong>
              Röviden átbeszéljük a panaszt és a célokat, majd egyeztetünk a
              vizsgálat/kezelés időpontjáról.
            </p>
          </div>
          <div class="benefit-card">
            <h3>Fókuszban a biztonságos, reális elvárások</h3>
            <p>
              <strong>Orvosi ígéretek nélkül, tiszta kommunikációval.</strong>
              A cél a mozgásminőség javítása és a panaszok csökkentése. Minden
              lépést közérthetően elmagyarázunk, és szükség esetén otthoni
              javaslatokat is adunk.
            </p>
          </div>
        </div>

        <div class="intro-image-block">
          <NuxtImg
            src="/img/hero.svg"
            alt="Csontkovács, manuálterápia és oszteopátia"
            class="intro-image"
            width="520"
            height="260"
          />
        </div>

        <!-- HOGYAN MŰKÖDIK SECTION -->
        <div class="process-section">
          <h2 class="section-heading">
            Hogyan zajlik a jelentkezés és a kezelés?
          </h2>

          <div class="process-progress" aria-label="Haladás a lépések között">
            <div class="process-progress__bar" aria-hidden="true">
              <div
                class="process-progress__fill"
                :style="{ width: ((activeProcessStep - 1) / 3) * 100 + '%' }"
              ></div>
            </div>
            <div class="process-progress__dots">
              <button
                v-for="n in 4"
                :key="n"
                type="button"
                class="process-progress__dot"
                :class="{
                  'is-active': activeProcessStep === n,
                  'is-done': activeProcessStep > n,
                }"
                @click="
                  processStepRefs?.[n - 1]?.scrollIntoView({
                    behavior: 'smooth',
                    block: 'center',
                  })
                "
                :aria-label="`Ugrás a(z) ${n}. lépéshez`"
              >
                <span class="process-progress__dot-number">{{ n }}</span>
              </button>
            </div>
            <div class="process-progress__label">
              Lépés {{ activeProcessStep }} / 4
            </div>
          </div>

          <div class="process-steps">
            <div
              class="process-step"
              :class="{
                'is-active': activeProcessStep === 1,
                'is-done': activeProcessStep > 1,
              }"
              data-step="1"
              :ref="(el) => (processStepRefs[0] = el)"
            >
              <div class="step-number">1</div>
              <div class="step-content">
                <h4>Bejelentkezés és előzetes egyeztetés</h4>
                <p>
                  Röviden átbeszéljük a fő panaszt és az elvárásokat, majd
                  időpontot egyeztetünk.
                </p>
              </div>
            </div>
            <div
              class="process-step"
              :class="{
                'is-active': activeProcessStep === 2,
                'is-done': activeProcessStep > 2,
              }"
              data-step="2"
              :ref="(el) => (processStepRefs[1] = el)"
            >
              <div class="step-number">2</div>
              <div class="step-content">
                <h4>Állapotfelmérés és funkcionális vizsgálat</h4>
                <p>
                  Megnézzük a mozgástartományt, a terhelhetőséget és az
                  összefüggéseket (akár a teljes testben), hogy célzottan
                  válasszunk technikát.
                </p>
              </div>
            </div>
            <div
              class="process-step"
              :class="{
                'is-active': activeProcessStep === 3,
                'is-done': activeProcessStep > 3,
              }"
              data-step="3"
              :ref="(el) => (processStepRefs[2] = el)"
            >
              <div class="step-number">3</div>
              <div class="step-content">
                <h4>Személyre szabott kezelési terv</h4>
                <p>
                  Átbeszéljük a várható menetet, a fókuszterületeket és a
                  javasolt lépéseket. Ezután kezdődik a manuális/csontkovács
                  kezelés.
                </p>
              </div>
            </div>
            <div
              class="process-step"
              :class="{
                'is-active': activeProcessStep === 4,
                'is-done': activeProcessStep > 4,
              }"
              data-step="4"
              :ref="(el) => (processStepRefs[3] = el)"
            >
              <div class="step-number">4</div>
              <div class="step-content">
                <h4>Kezelés + otthoni javaslatok</h4>
                <p>
                  A kezelés után röviden visszamérünk, és szükség esetén
                  egyszerű, otthon is végezhető javaslatokat adunk a javulás
                  támogatására.
                </p>
              </div>
            </div>
          </div>
        </div>

        <!-- GYIK Szekció -->
        <div class="faq-section">
          <h2 class="section-heading">Gyakran feltett kérdések</h2>

          <div class="faq-list">
            <details
              class="faq-item"
              :ref="(el) => (faqRefs[0] = el)"
              @toggle="handleFaqToggle(0)"
            >
              <summary class="faq-question">
                <span class="faq-title"
                  >🏆 Milyen engedéllyel rendelkeznek?</span
                >
                <span class="faq-icon" aria-hidden="true"></span>
              </summary>
              <div class="faq-answer">
                <p class="faq-text">
                  A kezelés minden esetben állapotfelmérésre épül, és az
                  aktuális terhelhetőségéhez igazítjuk. A cél a biztonságos,
                  fokozatos javítás – túlzó ígéretek nélkül.
                </p>
              </div>
            </details>

            <details
              class="faq-item"
              :ref="(el) => (faqRefs[2] = el)"
              @toggle="handleFaqToggle(2)"
            >
              <summary class="faq-question">
                <span class="faq-title">⏱️ Mennyi idő alatt érkezik ki?</span>
                <span class="faq-icon" aria-hidden="true"></span>
              </summary>
              <div class="faq-answer">
                <p class="faq-text">
                  Jellemzően néhány napon belül tudunk időpontot adni, a pontos
                  lehetőségek a terheltségtől függenek. Telefonon és űrlapon is
                  tud egyeztetni.
                </p>
              </div>
            </details>

            <details
              class="faq-item"
              :ref="(el) => (faqRefs[3] = el)"
              @toggle="handleFaqToggle(3)"
            >
              <summary class="faq-question">
                <span class="faq-title">🔧 Vállalnak kisebb munkákat is?</span>
                <span class="faq-icon" aria-hidden="true"></span>
              </summary>
              <div class="faq-answer">
                <p class="faq-text">
                  Igen. A kezelések gyakori okai: derék- és hátfájdalom,
                  nyak/vállfeszülés, ízületi beszűkülések, testtartási
                  problémák, valamint sportból vagy ülőmunkából adódó
                  túlterhelések.
                </p>
              </div>
            </details>

            <details
              class="faq-item"
              :ref="(el) => (faqRefs[4] = el)"
              @toggle="handleFaqToggle(4)"
            >
              <summary class="faq-question">
                <span class="faq-title">✅ Van garancia a munkákra?</span>
                <span class="faq-icon" aria-hidden="true"></span>
              </summary>
              <div class="faq-answer">
                <p class="faq-text">
                  A manuális kezelés eredménye egyénenként eltérő, ezért
                  "garantált" gyógyulást nem ígérünk. Amit biztosan kap: alapos
                  felmérést, tiszta tájékoztatást és személyre szabott kezelést.
                </p>
              </div>
            </details>

            <details
              class="faq-item"
              :ref="(el) => (faqRefs[5] = el)"
              @toggle="handleFaqToggle(5)"
            >
              <summary class="faq-question">
                <span class="faq-title"
                  >🏢 Milyen szolgáltatásokat nyújtanak?</span
                >
                <span class="faq-icon" aria-hidden="true"></span>
              </summary>
              <div class="faq-answer">
                <p class="faq-text">
                  Csontkovács kezelések, ízületi mobilizáció, izom- és fascia
                  technikák, gerinc és ízületi blokkok oldása, valamint
                  speciális területek: TMJ (állkapocs) és zsigeri (viszcerális)
                  oszteopátiás szemléletű kezelés.
                </p>
              </div>
            </details>
          </div>
        </div>

        <h2 class="section-heading">
          Ingyenes előzetes egyeztetés vagy időpontkérés
        </h2>

        <!-- CONTACT METHOD CHOICE -->
        <div class="contact-choice">
          <p class="choice-intro">
            Válasszon: hívjon minket, vagy kérjen időpontot az űrlapon!
          </p>

          <div class="choice-buttons">
            <button
              class="choice-btn choice-btn--phone"
              :class="{ active: contactMethod === 'phone' }"
              @click="contactMethod = 'phone'"
            >
              <span class="choice-icon">☎️</span>
              <span class="choice-text">Hívjon minket most!</span>
            </button>

            <button
              class="choice-btn choice-btn--form"
              :class="{ active: contactMethod === 'form' }"
              @click="contactMethod = 'form'"
            >
              <span class="choice-icon">📝</span>
              <span class="choice-text">Töltse ki az űrlapot</span>
            </button>
          </div>
        </div>

        <!-- PHONE CARD - Conditional -->
        <div v-if="contactMethod === 'phone'" class="contact-section">
          <div class="phone-card">
            <div class="phone-card-icon">☎️</div>
            <h3 class="phone-card-title">Hívjon minket most!</h3>
            <p class="phone-card-subtitle">
              Egyeztessen közvetlenül a kezelésről és az időpontról
            </p>

            <a href="tel:+36705985701" class="phone-button">
              <span class="phone-icon">📞</span>
              <span class="phone-number">06 70 598 5701</span>
            </a>

            <div class="phone-info">
              <p><strong>Időpontfoglalás és tájékoztatás telefonon</strong></p>
              <p class="small">Hívás esetén visszahívást is kérhet</p>
            </div>

            <div class="phone-benefits">
              <div class="benefit">✓ Azonnali válasz</div>
              <div class="benefit">✓ Rövid, közérthető tájékoztatás</div>
              <div class="benefit">✓ Könnyű egyeztetés</div>
            </div>
          </div>
        </div>

        <!-- FORM - Conditional -->
        <div v-if="contactMethod === 'form'" class="contact-section">
          <form class="appointment-form" @submit="submitForm">
            <!-- Szolgáltatás adatok -->
            <div class="form-section">
              <h3 class="section-title">Miben tudunk segíteni?</h3>

              <div class="form-group">
                <label class="supage-content__ul__li__strong"
                  >Szolgáltatás típusa *</label
                >
                <select
                  v-model="formData.serviceType"
                  required
                  class="form-select"
                  :disabled="isSubmitting"
                >
                  <option value="">Válasszon szolgáltatást...</option>
                  <option value="csontkovacs">
                    Csontkovács kezelés (gerinc / ízületek)
                  </option>
                  <option value="manualterapia">
                    Manuálterápia (izom, fascia, mobilizáció)
                  </option>
                  <option value="derek_hat">Derék- és hátfájdalom</option>
                  <option value="nyak_vall">Nyak- és vállfeszülés</option>
                  <option value="testtartas">Testtartási problémák</option>
                  <option value="tmj">TMJ / állkapocs panaszok</option>
                  <option value="zsigeri">
                    Zsigeri (viszcerális) oszteopátia
                  </option>
                  <option value="sport">
                    Sport / aktív életmód melletti panasz
                  </option>
                  <option value="egyeb">Egyéb / nem biztos benne</option>
                </select>
              </div>

              <input
                type="hidden"
                v-model="formData.propertyType"
                :disabled="isSubmitting"
              />

              <div class="form-group">
                <label class="supage-content__ul__li__strong"
                  >Milyen sürgős az igény? *</label
                >
                <select
                  v-model="formData.urgency"
                  required
                  class="form-select"
                  :disabled="isSubmitting"
                >
                  <option value="">Válasszon sürgetőséget...</option>
                  <option value="sürgős">Sürgős (ma még szükséges)</option>
                  <option value="rövid">Rövid időn belül (1-2 nap)</option>
                  <option value="normál">Normál (1-2 hét)</option>
                </select>
              </div>
            </div>

            <!-- Személyes adatok -->
            <div class="form-section">
              <h3 class="section-title">Személyes adatok</h3>

              <div class="form-group">
                <label class="supage-content__ul__li__strong">Név *</label>
                <input
                  type="text"
                  v-model="formData.name"
                  required
                  class="form-input"
                  placeholder="Teljes név"
                  :disabled="isSubmitting"
                />
              </div>

              <div class="form-group">
                <label class="supage-content__ul__li__strong"
                  >Email cím *</label
                >
                <input
                  type="email"
                  v-model="formData.email"
                  required
                  class="form-input"
                  placeholder="Az időpont egyeztetéshez ide írjon email címet"
                  :disabled="isSubmitting"
                />
              </div>

              <div class="form-group">
                <label class="supage-content__ul__li__strong"
                  >Telefonszám *</label
                >
                <input
                  type="tel"
                  v-model="formData.phone"
                  required
                  class="form-input"
                  placeholder="Gyors egyeztetéshez"
                  :disabled="isSubmitting"
                />
              </div>

              <div class="form-group">
                <label class="supage-content__ul__li__strong"
                  >Cím / Helyszín *</label
                >
                <input
                  type="text"
                  v-model="formData.address"
                  required
                  class="form-input"
                  placeholder="Budapest (kerület) / rendelő megközelítése miatt"
                  :disabled="isSubmitting"
                />
              </div>
            </div>

            <!-- Megjegyzés -->
            <div class="form-section">
              <h3 class="section-title">Megjegyzés (Opcionális)</h3>

              <div class="form-group">
                <label class="supage-content__ul__li__strong"
                  >További információ</label
                >
                <textarea
                  v-model="formData.message"
                  class="form-textarea"
                  placeholder="Röviden írja le a panaszt (mióta tart, mi rontja/javítja), és ha szeretné, írjon lehetséges időpontokat is."
                  rows="4"
                  :disabled="isSubmitting"
                ></textarea>
              </div>
            </div>

            <button type="submit" class="submit-btn" :disabled="isSubmitting">
              <span class="btn-text" v-if="!isSubmitting"
                >Időpontkérés elküldése</span
              >
              <span class="btn-text" v-else>Küldés...</span>
            </button>

            <p class="privacy-text">
              <i class="supage-content__p__i">
                Az időpontkérés elküldésével automatikusan elfogadja az
                <NuxtLink
                  class="supage-content__nlink"
                  to="/adatvedelmi-tajekoztato"
                  >Adatvédelmi Szabályzatot.</NuxtLink
                >
              </i>
            </p>
          </form>

          <!-- Success/Error Message -->
          <div
            v-if="submitMessage"
            class="submit-message"
            :class="{
              success: submitMessage.includes('✅'),
              error: submitMessage.includes('❌'),
            }"
          >
            {{ submitMessage }}
          </div>
        </div>
      </div>
    </div>
  </section>
</template>

<style scoped>
/* Header és footer elrejtése */
header,
footer,
.header,
.footer,
nav,
.navbar,
.site-header,
.site-footer {
  display: none !important;
}

body > header,
body > footer {
  display: none !important;
}

.subpage-content {
  padding: 3em;
  max-width: 1200px;
  margin: 0 auto;
  width: 100%;
  background-color: #f2f2f2;
}

/* ========== HERO BANNER ========== */
.trust-banner {
  position: relative;
  border-radius: 15px;
  margin-bottom: 40px;
  overflow: hidden;
  box-shadow: 0 10px 30px rgba(17, 49, 115, 0.25);
}

.trust-banner--with-image {
  background: #fff;
}

.banner-bg-image {
  position: relative;
  width: 100%;
  height: auto;
  overflow: hidden;
  z-index: 1;
  display: block;
}

.banner-image {
  width: 100%;
  height: auto;
  object-fit: cover;
  object-position: center;
  display: block;
  opacity: 1;
}

.banner-content {
  position: relative;
  z-index: 3;
  text-align: center;
  padding: 3.5em 2em;
  width: 100%;
  background: linear-gradient(180deg, #1e40af 0%, #102a67 100%);
}

.main-title {
  font-size: 2.2rem;
  font-weight: bold;
  margin-bottom: 10px;
  text-align: center;
  color: #fff;
  text-shadow: 0 2px 4px rgba(0, 0, 0, 0.2);
  letter-spacing: 0.5px;
}

.banner-subtitle {
  font-size: 1.1rem;
  text-align: center;
  color: #fff;
  margin-bottom: 0;
  opacity: 0.95;
  text-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
}

/* ========== 3 KIEMELT ÉRVELÉS ========== */
.benefits-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(280px, 1fr));
  gap: 25px;
  margin: 45px 0 50px 0;
  padding: 0;
}

.benefit-card {
  background: #fff;
  padding: 28px;
  border-radius: 10px;
  border-left: 6px solid #1e40af;
  box-shadow: 0 4px 15px rgba(17, 49, 115, 0.08);
  transition: all 0.3s ease;
}

.benefit-card:hover {
  transform: translateY(-5px);
  box-shadow: 0 8px 25px rgba(17, 49, 115, 0.15);
  background: #f9f9f9;
}

.benefit-card h3 {
  color: #1e40af;
  font-size: 1.15rem;
  margin-bottom: 15px;
  font-weight: 700;
  line-height: 1.3;
}

.benefit-card p {
  color: #333;
  line-height: 1.6;
  margin: 0;
  font-size: 0.95rem;
}

.benefit-card strong {
  color: #000;
  font-weight: 700;
}

/* ========== SECTION HEADING ========== */
.section-heading {
  color: #1e40af;
  font-size: 2rem;
  font-weight: bold;
  margin: 1.5em 0 1.5em 0;
  text-align: center;
}

/* ========== PROCESS / LÉPÉSEK (DOPAMIN PROGRESS) ========== */
.process-section {
  margin: 0 auto 50px auto;
  max-width: 1000px;
}

.process-progress {
  position: sticky;
  top: 16px;
  z-index: 5;
  margin: 0 auto 18px auto;
  padding: 14px 16px;
  border-radius: 14px;
  background: linear-gradient(
    180deg,
    rgba(244, 248, 255, 0.95) 0%,
    rgba(255, 255, 255, 0.92) 100%
  );
  backdrop-filter: blur(10px);
  border: 1px solid rgba(17, 49, 115, 0.14);
  box-shadow: 0 10px 30px rgba(17, 49, 115, 0.12);
}

.process-progress__bar {
  height: 10px;
  border-radius: 999px;
  background: linear-gradient(
    90deg,
    rgba(19, 52, 117, 0.12),
    rgba(19, 52, 117, 0.06)
  );
  overflow: hidden;
}

.process-progress__fill {
  height: 100%;
  border-radius: 999px;
  background: linear-gradient(90deg, #1e40af 0%, #2f80ed 55%, #0ea5a4 100%);
  width: 0%;
  transition: width 450ms cubic-bezier(0.2, 0.9, 0.2, 1);
}

.process-progress__dots {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 10px;
  margin-top: 12px;
  align-items: center;
}

.process-progress__dot {
  width: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 10px 0;
  border-radius: 12px;
  border: 1px solid rgba(17, 49, 115, 0.18);
  background: #fff;
  color: #1e40af;
  cursor: pointer;
  transition:
    transform 180ms ease,
    box-shadow 220ms ease,
    background 220ms ease,
    border-color 220ms ease;
}

.process-progress__dot:hover {
  transform: translateY(-1px);
  box-shadow: 0 8px 18px rgba(17, 49, 115, 0.16);
  border-color: rgba(17, 49, 115, 0.35);
}

.process-progress__dot.is-done {
  background: rgba(14, 165, 164, 0.09);
  border-color: rgba(14, 165, 164, 0.35);
  color: #0f766e;
}

.process-progress__dot.is-active {
  background: linear-gradient(135deg, #1e40af 0%, #2f80ed 100%);
  color: #fff;
  border-color: transparent;
  box-shadow: 0 12px 26px rgba(17, 49, 115, 0.25);
  transform: translateY(-1px);
}

.process-progress__dot-number {
  font-weight: 800;
  letter-spacing: 0.2px;
}

.process-progress__label {
  margin-top: 10px;
  text-align: center;
  font-weight: 700;
  font-size: 0.95rem;
  color: #102a67;
}

.process-steps {
  display: grid;
  gap: 14px;
}

.process-step {
  background: linear-gradient(180deg, #ffffff 0%, #f4f8ff 100%);
  border-radius: 14px;
  border: 1px solid rgba(17, 49, 115, 0.12);
  box-shadow: 0 6px 22px rgba(17, 49, 115, 0.08);
  padding: 18px 18px;
  display: grid;
  grid-template-columns: 52px 1fr;
  gap: 14px;
  transition:
    transform 180ms ease,
    box-shadow 220ms ease,
    border-color 220ms ease;
}

.process-step.is-active {
  border-color: rgba(47, 128, 237, 0.55);
  box-shadow: 0 16px 40px rgba(17, 49, 115, 0.16);
  transform: translateY(-1px);
}

.process-step.is-done {
  border-color: rgba(14, 165, 164, 0.35);
}

.step-number {
  width: 44px;
  height: 44px;
  border-radius: 999px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-weight: 900;
  color: #1e40af;
  background: rgba(17, 49, 115, 0.08);
  border: 1px solid rgba(17, 49, 115, 0.16);
}

.process-step.is-active .step-number {
  color: #fff;
  background: linear-gradient(135deg, #1e40af 0%, #2f80ed 100%);
  border-color: transparent;
  box-shadow: 0 10px 22px rgba(17, 49, 115, 0.25);
}

.process-step.is-done .step-number {
  color: #0f766e;
  background: rgba(14, 165, 164, 0.14);
  border-color: rgba(14, 165, 164, 0.35);
}

.step-content h4 {
  margin: 2px 0 6px 0;
  font-size: 1.05rem;
  color: #102a67;
  font-weight: 800;
}

.step-content p {
  margin: 0;
  color: #333;
  line-height: 1.6;
  font-size: 0.95rem;
}

@media screen and (max-width: 767px) {
  .process-progress {
    position: relative;
    top: auto;
    padding: 12px 12px;
  }

  .process-step {
    grid-template-columns: 44px 1fr;
    padding: 16px 14px;
  }

  .step-number {
    width: 38px;
    height: 38px;
  }
}

/* ========== CONTACT CHOICE ========== */
.contact-choice {
  background: #fff;
  padding: 40px 30px;
  border-radius: 15px;
  text-align: center;
  margin: 2.5em auto;
  max-width: 900px;
  box-shadow: 0 5px 20px rgba(19, 52, 117, 0.1);
  border: 2px solid #e0e0e0;
}

.choice-intro {
  color: #1e40af;
  font-size: 1.3rem;
  font-weight: 700;
  margin: 0 0 30px 0;
}

.choice-buttons {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 20px;
}

.choice-btn {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 12px;
  padding: 25px 20px;
  border: 3px solid #ddd;
  background: #fff;
  border-radius: 12px;
  cursor: pointer;
  transition: all 0.3s ease;
  font-size: 1rem;
  font-weight: 600;
  color: #333;
}

.choice-btn:hover {
  border-color: #1e40af;
  transform: translateY(-3px);
  box-shadow: 0 8px 20px rgba(17, 49, 115, 0.15);
}

.choice-btn.active {
  background: linear-gradient(135deg, #1e40af 0%, #102a67 100%);
  color: #fff;
  border-color: #1e40af;
  box-shadow: 0 8px 25px rgba(17, 49, 115, 0.25);
}

.choice-icon {
  font-size: 2rem;
}

.choice-text {
  text-align: center;
  line-height: 1.3;
}

/* ========== CONTACT SECTION ========== */
.contact-section {
  margin: 2em auto;
  max-width: 900px;
  animation: fadeIn 0.3s ease-in;
}

@keyframes fadeIn {
  from {
    opacity: 0;
    transform: translateY(10px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

/* ========== PHONE CARD ========== */
.phone-card {
  background: linear-gradient(135deg, #1e40af 0%, #102a67 100%);
  padding: 45px 35px;
  border-radius: 15px;
  text-align: center;
  box-shadow: 0 10px 30px rgba(17, 49, 115, 0.2);
  color: #fff;
}

.phone-card-icon {
  font-size: 3.5rem;
  margin-bottom: 15px;
}

.phone-card-title {
  font-size: 1.6rem;
  font-weight: 700;
  margin: 0 0 8px 0;
  color: #fff;
}

.phone-card-subtitle {
  color: rgba(255, 255, 255, 0.9);
  font-size: 1rem;
  margin: 0 0 30px 0;
}

.phone-button {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: 12px;
  background: #fff;
  color: #1e40af;
  padding: 18px 35px;
  border-radius: 50px;
  text-decoration: none;
  font-weight: 700;
  font-size: 1.15rem;
  margin: 0 0 30px 0;
  transition: all 0.3s ease;
  box-shadow: 0 6px 15px rgba(0, 0, 0, 0.15);
  border: none;
  cursor: pointer;
}

.intro-image-block {
  display: flex;
  justify-content: center;
  margin: 10px 0 45px 0;
}

.intro-image {
  width: 100%;
  max-width: 520px;
  height: auto;
  border-radius: 12px;
  box-shadow: 0 10px 30px rgba(17, 49, 115, 0.16);
}

.phone-button:hover {
  transform: translateY(-3px);
  box-shadow: 0 10px 30px rgba(0, 0, 0, 0.25);
  background: #f0f0f0;
}

.phone-button:active {
  transform: translateY(-1px);
}

.phone-icon {
  font-size: 1.4rem;
}

.phone-number {
  letter-spacing: 0.5px;
}

.phone-info {
  background: rgba(255, 255, 255, 0.15);
  padding: 18px 16px;
  border-radius: 10px;
  margin-bottom: 25px;
}

.phone-info p {
  margin: 0;
  font-size: 0.95rem;
  line-height: 1.5;
}

.phone-info p strong {
  display: block;
  font-weight: 700;
}

.phone-info p.small {
  font-size: 0.85rem;
  opacity: 0.9;
  margin-top: 6px;
}

.phone-benefits {
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.phone-benefits .benefit {
  font-size: 0.95rem;
  line-height: 1.4;
  color: #fff;
}

/* ========== FORM STYLES ========== */
.appointment-form {
  max-width: 900px;
  margin: 0 auto;
}

.submit-message {
  padding: 18px;
  border-radius: 10px;
  margin-top: 20px;
  font-weight: bold;
  text-align: center;
  font-size: 1rem;
}

.submit-message.success {
  background-color: #e6fffb;
  color: #0f766e;
  border: 2px solid #99f6e4;
}

.submit-message.error {
  background-color: #fee2e2;
  color: #991b1b;
  border: 2px solid #fecaca;
}

.form-section {
  background: #fff;
  padding: 28px;
  margin: 2.5em 0;
  border-radius: 12px;
  box-shadow: 0 3px 12px rgba(17, 49, 115, 0.08);
  border-left: 5px solid #1e40af;
}

.section-title {
  color: #1e40af;
  font-size: 1.35rem;
  margin-bottom: 25px;
  border-bottom: 3px solid #1e40af;
  padding-bottom: 12px;
  font-weight: 700;
}

.form-group {
  margin-bottom: 22px;
}

.form-group label {
  display: block;
  margin-bottom: 8px;
  color: #333;
  font-weight: 600;
  font-size: 0.95rem;
}

.form-input,
.form-textarea {
  width: 100%;
  padding: 13px 15px;
  border: 2px solid #ddd;
  border-radius: 8px;
  font-size: 15px;
  font-family: inherit;
  transition: all 0.3s;
  background-color: #fff;
  color: #333;
}

.form-input::placeholder,
.form-textarea::placeholder {
  color: #999;
}

.form-select {
  width: 100%;
  padding: 13px 16px;
  border: 2px solid #ddd;
  border-radius: 8px;
  font-size: 15px;
  font-family: inherit;
  background-color: #fff;
  transition: all 0.3s ease;
  cursor: pointer;
  color: #333;
  -webkit-appearance: none;
  -moz-appearance: none;
  appearance: none;
  background-image: url("data:image/svg+xml;charset=UTF-8,%3csvg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 24 24' fill='none' stroke='%231e40af' stroke-width='2.5' stroke-linecap='round' stroke-linejoin='round'%3e%3cpolyline points='6,9 12,15 18,9'%3e%3c/polyline%3e%3c/svg%3e");
  background-repeat: no-repeat;
  background-position: right 12px center;
  background-size: 18px;
  padding-right: 45px;
}

.form-input:focus,
.form-textarea:focus {
  border-color: #1e40af;
  outline: none;
  box-shadow: 0 0 0 4px rgba(17, 49, 115, 0.1);
}

.form-select:focus {
  border-color: #1e40af;
  outline: none;
  box-shadow: 0 0 0 4px rgba(17, 49, 115, 0.1);
}

.form-select:hover:not(:disabled) {
  border-color: #1e40af;
  background-color: #fefefe;
}

.form-input:disabled,
.form-textarea:disabled,
.form-select:disabled {
  background-color: #f0f0f0;
  opacity: 0.65;
  cursor: not-allowed;
}

/* ========== SUBMIT BUTTON ========== */
.submit-btn {
  background: linear-gradient(135deg, #1e40af 0%, #102a67 100%);
  color: #fff;
  border: none;
  padding: 18px 45px;
  border-radius: 10px;
  font-size: 1.1rem;
  font-weight: 700;
  cursor: pointer;
  width: 100%;
  transition: all 0.3s ease;
  box-shadow: 0 6px 20px rgba(17, 49, 115, 0.3);
  margin-top: 15px;
}

.submit-btn:hover:not(:disabled) {
  transform: translateY(-3px);
  box-shadow: 0 10px 30px rgba(17, 49, 115, 0.4);
  background: linear-gradient(135deg, #102a67 0%, #1e40af 100%);
}

.submit-btn:active:not(:disabled) {
  transform: translateY(-1px);
}

.submit-btn:disabled {
  opacity: 0.65;
  cursor: not-allowed;
  transform: none;
}

.btn-text {
  font-size: 1.05rem;
  letter-spacing: 0.5px;
}

/* ========== PRIVACY TEXT ========== */
.privacy-text {
  color: #666;
  font-size: 0.85rem;
  margin-top: 15px;
  text-align: center;
}

.privacy-text a {
  color: #1e40af;
  text-decoration: none;
  font-weight: 600;
  transition: color 0.3s;
}

.privacy-text a:hover {
  color: #102a67;
  text-decoration: underline;
}

/* ========== FAQ SECTION ========== */
.faq-section {
  margin: 3em 0;
}

.faq-list {
  display: flex;
  flex-direction: column;
  gap: 18px;
  margin-top: 30px;
}

.faq-item {
  background: #fff;
  border-radius: 10px;
  border-left: 5px solid #1e40af;
  box-shadow: 0 3px 12px rgba(17, 49, 115, 0.08);
  transition: all 0.25s ease;
  overflow: hidden;
  width: 100%;
}

.faq-item:hover {
  transform: translateY(-2px);
  box-shadow: 0 6px 20px rgba(17, 49, 115, 0.12);
  background: #f9f9f9;
}

.faq-question {
  list-style: none;
  cursor: pointer;
  padding: 20px 22px;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 14px;
}

.faq-question::-webkit-details-marker {
  display: none;
}

.faq-title {
  color: #1e40af;
  font-size: 1.05rem;
  font-weight: 700;
  line-height: 1.4;
}

.faq-icon {
  width: 18px;
  height: 18px;
  flex: 0 0 18px;
  position: relative;
}

.faq-icon::before,
.faq-icon::after {
  content: '';
  position: absolute;
  top: 50%;
  left: 50%;
  width: 11px;
  height: 2px;
  background: #1e40af;
  transform: translate(-50%, -50%);
  border-radius: 2px;
  transition: transform 0.2s ease;
}

.faq-icon::after {
  transform: translate(-50%, -50%) rotate(90deg);
}

.faq-item[open] .faq-icon::after {
  transform: translate(-50%, -50%) rotate(0deg);
}

.faq-item[open] .faq-question {
  border-bottom: 1px solid rgba(17, 49, 115, 0.12);
}

.faq-answer {
  padding: 17px 22px 18px 22px;
}

.faq-text {
  color: #333;
  font-size: 0.9rem;
  line-height: 1.6;
  margin: 0;
}

/* ========== RESPONSIVE ========== */
@media (max-width: 768px) {
  .choice-buttons {
    grid-template-columns: 1fr;
  }

  .choice-btn {
    padding: 20px 15px;
  }

  .phone-card {
    padding: 30px 25px;
  }

  .phone-card-icon {
    font-size: 2.5rem;
  }

  .phone-button {
    padding: 16px 28px;
    font-size: 1rem;
  }

  .main-title {
    font-size: 1.8rem;
  }

  .banner-subtitle {
    font-size: 0.95rem;
  }

  .banner-content {
    padding: 2.5em 1.5em;
  }

  .benefits-grid {
    grid-template-columns: 1fr;
  }

  .benefit-card {
    padding: 20px;
  }

  .form-section {
    padding: 20px;
  }

  .section-heading {
    font-size: 1.6rem;
  }

  .section-title {
    font-size: 1.15rem;
  }

  .subpage-content {
    padding: 1.5em;
  }
}

@media (max-width: 480px) {
  .trust-banner {
    border-radius: 10px;
  }

  .main-title {
    font-size: 1.5rem;
    margin-bottom: 8px;
  }

  .banner-subtitle {
    font-size: 0.9rem;
  }

  .banner-content {
    padding: 2em 1em;
  }

  .benefits-grid {
    gap: 15px;
  }

  .benefit-card {
    padding: 15px;
  }

  .benefit-card h3 {
    font-size: 1rem;
  }

  .subpage-content {
    padding: 1em;
  }

  .section-heading {
    font-size: 1.3rem;
  }

  .faq-title {
    font-size: 0.95rem;
  }

  .faq-text {
    font-size: 0.85rem;
  }

  .contact-choice {
    padding: 25px 20px;
  }

  .choice-intro {
    font-size: 1.1rem;
  }
}
</style>
