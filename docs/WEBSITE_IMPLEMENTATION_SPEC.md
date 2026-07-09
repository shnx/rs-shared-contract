# Website Implementation Spec — Consultation Ecosystem

**Date:** July 9, 2026
**From:** iOS team
**To:** Web team
**Purpose:** Exact page layouts, copy, and component structure for the minimum viable consultation ecosystem website.

---

## Design System (Keep Existing Apple Theme)

The website already uses an Apple-like theme. **Do not change the overall look.** Keep:
- White backgrounds, gray-50 sections
- gray-900 buttons
- Inter / system font
- Rounded corners (rounded-lg, rounded-xl)
- Subtle borders (border-gray-100, border-gray-200)
- Lucide icons
- Generous whitespace

Only **add new pages** and **update copy** on existing ones.

---

## Page 1: Homepage (Update Existing `HomePage.tsx`)

### Hero Section (Replace Current)

```
Headline (large, bold, gray-900):
  "Build Better Robots."
  "Build Smarter AI."
  "Build the Future."

Subtext (gray-500, max-w-lg):
  "Helping students, startups, and companies turn ambitious ideas into real products."

Three Buttons (side by side, not stacked):
  [🎓 I'm Learning]  → /consultations/first-coffee
  [🚀 I'm Building]  → /consultations/mentoring
  [🏢 I'm Hiring]    → /consultations/company
```

### "Not Sure Where to Start?" Section (New — Below Hero)

```
Light gray-50 background, centered:

  "Not sure where to start?"
  "Sometimes all you need is one conversation."
  [Book First Coffee → €15]  → /consultations/first-coffee
```

### Remove Existing Sections

Remove:
- "Start your project" CTA (replaced by 3 buttons above)
- "Achieve Your Career Goals" with $25/$500 pricing (replaced by consultation pages)
- "Talent CTA" section (keep link in footer instead)

### Keep

- Animated rotating phrases hero (update phrases to match new headline if desired)

---

## Page 2: First Coffee (New Page — `/consultations/first-coffee`)

### Structure

```tsx
<div>
  {/* Hero */}
  <section className="py-20 bg-white">
    <div className="max-w-2xl mx-auto px-4 text-center">
      <div className="w-16 h-16 bg-amber-50 rounded-2xl flex items-center justify-center mx-auto mb-6">
        <Coffee className="h-8 w-8 text-amber-600" />
      </div>
      <h1 className="text-4xl md:text-5xl font-bold text-gray-900 tracking-tight mb-4">
        Sometimes one conversation changes everything.
      </h1>
      <p className="text-lg text-gray-500 leading-relaxed">
        This session exists because, throughout my career, many of the opportunities
        that shaped my life began with someone generous enough to share their experience.
        I wanted to create that same opportunity for others.
      </p>
    </div>
  </section>

  {/* What We Can Talk About */}
  <section className="py-16 bg-gray-50 border-t border-gray-100">
    <div className="max-w-2xl mx-auto px-4">
      <h2 className="text-2xl font-bold text-gray-900 mb-8 text-center">
        What we can talk about
      </h2>
      <div className="grid md:grid-cols-2 gap-3">
        {topics.map(topic => (
          <div className="flex items-center gap-3 bg-white rounded-xl border border-gray-100 p-4">
            <div className="w-8 h-8 bg-gray-50 rounded-lg flex items-center justify-center">
              <Check className="h-4 w-4 text-gray-600" />
            </div>
            <span className="text-sm text-gray-700">{topic}</span>
          </div>
        ))}
      </div>
    </div>
  </section>

  {/* Details & Booking */}
  <section className="py-16 bg-white border-t border-gray-100">
    <div className="max-w-md mx-auto px-4 text-center">
      <div className="flex items-center justify-center gap-6 mb-8">
        <div className="flex items-center gap-2 text-gray-600">
          <Clock className="h-5 w-5 text-gray-400" />
          <span className="text-sm">~30 minutes</span>
        </div>
        <div className="flex items-center gap-2">
          <span className="text-3xl font-bold text-gray-900">€15</span>
        </div>
      </div>
      <p className="text-sm text-gray-500 mb-6">
        Comparable to buying us both a coffee.
      </p>
      <Link to="/discover?type=first_coffee"
        className="inline-flex items-center gap-2 px-6 py-3 bg-gray-900 text-white rounded-lg font-medium hover:bg-black transition">
        Book First Coffee <ArrowRight className="h-4 w-4" />
      </Link>
    </div>
  </section>
</div>
```

### Topics Array

```ts
const topics = [
  "Career direction (AI vs Robotics)",
  "Master's applications",
  "CV review",
  "Technical interviews",
  "Building your first product",
  "Starting a robotics company",
  "Manufacturing, electronics, embedded systems",
  "Computer vision, ROS, AI",
  "Research ideas",
  "Or simply a challenge you've been thinking about",
];
```

---

## Page 3: Student Mentoring (New Page — `/consultations/mentoring`)

### Structure

```tsx
<div>
  {/* Hero */}
  <section className="py-20 bg-white">
    <div className="max-w-2xl mx-auto px-4 text-center">
      <div className="w-16 h-16 bg-emerald-50 rounded-2xl flex items-center justify-center mx-auto mb-6">
        <GraduationCap className="h-8 w-8 text-emerald-600" />
      </div>
      <h1 className="text-4xl md:text-5xl font-bold text-gray-900 tracking-tight mb-4">
        Build the engineer you want to become.
      </h1>
      <p className="text-lg text-gray-500 leading-relaxed">
        One meeting won't make you a robotics engineer. Consistent guidance will.
        Student mentoring is designed for people who want structured growth
        rather than isolated answers.
      </p>
    </div>
  </section>

  {/* Examples */}
  <section className="py-16 bg-gray-50 border-t border-gray-100">
    <div className="max-w-2xl mx-auto px-4">
      <h2 className="text-2xl font-bold text-gray-900 mb-8 text-center">
        How we can work together
      </h2>
      <div className="grid md:grid-cols-2 gap-3">
        {examples.map(item => (
          <div className="flex items-center gap-3 bg-white rounded-xl border border-gray-100 p-4">
            <div className="w-8 h-8 bg-gray-50 rounded-lg flex items-center justify-center">
              <Check className="h-4 w-4 text-gray-600" />
            </div>
            <span className="text-sm text-gray-700">{item}</span>
          </div>
        ))}
      </div>
    </div>
  </section>

  {/* Pricing & CTA */}
  <section className="py-16 bg-white border-t border-gray-100">
    <div className="max-w-md mx-auto px-4 text-center">
      <p className="text-sm text-gray-500 mb-2">Starting at</p>
      <div className="flex items-center justify-center gap-2 mb-6">
        <span className="text-4xl font-bold text-gray-900">€60</span>
        <span className="text-lg text-gray-400">– €120 / hour</span>
      </div>
      <p className="text-sm text-gray-500 mb-6">
        Includes preparation, action plan, resources, and recording.
      </p>
      <Link to="/discover?type=mentoring"
        className="inline-flex items-center gap-2 px-6 py-3 bg-gray-900 text-white rounded-lg font-medium hover:bg-black transition">
        Request Mentoring <ArrowRight className="h-4 w-4" />
      </Link>
    </div>
  </section>
</div>
```

### Examples Array

```ts
const examples = [
  "Career planning",
  "Weekly mentoring sessions",
  "Project reviews",
  "Portfolio building",
  "Research guidance",
  "Technical interview preparation",
  "ROS, Computer Vision, Embedded Systems",
  "Graduation projects",
];
```

---

## Page 4: Startup Advisory (New Page — `/consultations/startup`)

### Structure

```tsx
<div>
  {/* Hero */}
  <section className="py-20 bg-white">
    <div className="max-w-2xl mx-auto px-4 text-center">
      <div className="w-16 h-16 bg-blue-50 rounded-2xl flex items-center justify-center mx-auto mb-6">
        <Rocket className="h-8 w-8 text-blue-600" />
      </div>
      <h1 className="text-4xl md:text-5xl font-bold text-gray-900 tracking-tight mb-4">
        Build your startup on solid technical foundations.
      </h1>
      <p className="text-lg text-gray-500 leading-relaxed">
        Most startups don't fail because engineers are bad. They fail because
        early technical decisions become expensive later. Let's make those
        decisions together.
      </p>
    </div>
  </section>

  {/* Common Topics */}
  <section className="py-16 bg-gray-50 border-t border-gray-100">
    <div className="max-w-2xl mx-auto px-4">
      <h2 className="text-2xl font-bold text-gray-900 mb-8 text-center">
        Common topics
      </h2>
      <div className="grid md:grid-cols-2 gap-3">
        {topics.map(item => (
          <div className="flex items-center gap-3 bg-white rounded-xl border border-gray-100 p-4">
            <div className="w-8 h-8 bg-gray-50 rounded-lg flex items-center justify-center">
              <Check className="h-4 w-4 text-gray-600" />
            </div>
            <span className="text-sm text-gray-700">{item}</span>
          </div>
        ))}
      </div>
    </div>
  </section>

  {/* CTA — No upfront pricing */}
  <section className="py-16 bg-white border-t border-gray-100">
    <div className="max-w-md mx-auto px-4 text-center">
      <p className="text-sm text-gray-500 mb-6">
        Engagements are tailored to the stage and complexity of your startup.
      </p>
      <Link to="/contact?type=startup"
        className="inline-flex items-center gap-2 px-6 py-3 bg-gray-900 text-white rounded-lg font-medium hover:bg-black transition">
        Start the Conversation <ArrowRight className="h-4 w-4" />
      </Link>
    </div>
  </section>
</div>
```

### Topics Array

```ts
const topics = [
  "MVP architecture",
  "Hardware selection",
  "AI integration",
  "Manufacturing strategy",
  "Technical hiring",
  "Investor technical preparation",
  "Product roadmap",
  "Automation",
  "Scaling",
  "Technical feasibility",
];
```

---

## Page 5: Company Advisory (New Page — `/consultations/company`)

### Structure

```tsx
<div>
  {/* Hero */}
  <section className="py-20 bg-white">
    <div className="max-w-2xl mx-auto px-4 text-center">
      <div className="w-16 h-16 bg-indigo-50 rounded-2xl flex items-center justify-center mx-auto mb-6">
        <Building2 className="h-8 w-8 text-indigo-600" />
      </div>
      <h1 className="text-4xl md:text-5xl font-bold text-gray-900 tracking-tight mb-4">
        Technical expertise when you need it most.
      </h1>
      <p className="text-lg text-gray-500 leading-relaxed">
        Whether you're evaluating robotics opportunities, integrating AI, or
        exploring automation, I provide independent technical guidance to help
        your team move with confidence.
      </p>
    </div>
  </section>

  {/* Ways to Work Together */}
  <section className="py-16 bg-gray-50 border-t border-gray-100">
    <div className="max-w-2xl mx-auto px-4">
      <h2 className="text-2xl font-bold text-gray-900 mb-8 text-center">
        Ways to work together
      </h2>
      <div className="grid md:grid-cols-2 gap-3">
        {ways.map(item => (
          <div className="flex items-center gap-3 bg-white rounded-xl border border-gray-100 p-4">
            <div className="w-8 h-8 bg-gray-50 rounded-lg flex items-center justify-center">
              <Check className="h-4 w-4 text-gray-600" />
            </div>
            <span className="text-sm text-gray-700">{item}</span>
          </div>
        ))}
      </div>
    </div>
  </section>

  {/* CTA — No upfront pricing */}
  <section className="py-16 bg-white border-t border-gray-100">
    <div className="max-w-md mx-auto px-4 text-center">
      <p className="text-sm text-gray-500 mb-6">
        Every engagement begins with a short discovery conversation to understand
        your goals before defining the most appropriate scope.
      </p>
      <Link to="/contact?type=company"
        className="inline-flex items-center gap-2 px-6 py-3 bg-gray-900 text-white rounded-lg font-medium hover:bg-black transition">
        Contact <ArrowRight className="h-4 w-4" />
      </Link>
    </div>
  </section>
</div>
```

### Ways Array

```ts
const ways = [
  "Technical Advisory",
  "Fractional CTO",
  "Architecture Reviews",
  "AI Strategy",
  "Robotics Consulting",
  "Technical Workshops",
  "Engineering Reviews",
  "Product Development",
  "Manufacturing Strategy",
];
```

---

## Page 6: Why Work With Me (New Section — Add to About Page or Homepage)

```tsx
<section className="py-20 bg-gray-50 border-t border-gray-100">
  <div className="max-w-4xl mx-auto px-4">
    <h2 className="text-2xl md:text-3xl font-bold text-gray-900 mb-8 text-center">
      Why work with me
    </h2>
    <div className="grid md:grid-cols-2 gap-4">
      {facts.map((fact, i) => (
        <div key={i} className="flex items-start gap-3 bg-white rounded-xl border border-gray-100 p-6">
          <div className="w-10 h-10 bg-gray-50 rounded-xl flex items-center justify-center flex-shrink-0">
            <fact.icon className="h-5 w-5 text-gray-600" />
          </div>
          <p className="text-sm text-gray-700 leading-relaxed">{fact.text}</p>
        </div>
      ))}
    </div>
  </div>
</section>
```

### Facts Array

```ts
const facts = [
  { icon: GraduationCap, text: "Master's degree in Robotics earned in Germany" },
  { icon: Rocket, text: "Founder building ADY, a robotics company focused on AI-powered camera systems" },
  { icon: Cpu, text: "Experience spanning robotics, embedded systems, electronics, AI, and computer vision" },
  { icon: Users, text: "Designed and delivered robotics and AI training programs for students and professionals" },
  { icon: Heart, text: "Mentor to students, engineers, and founders navigating technical and career decisions" },
];
```

---

## Footer Update (Update Existing `Footer.tsx`)

Add a "Where would you like to go next?" section above the existing footer:

```tsx
{/* Pre-footer navigation */}
<section className="py-12 bg-white border-t border-gray-100">
  <div className="max-w-4xl mx-auto px-4 text-center">
    <h3 className="text-lg font-semibold text-gray-900 mb-6">
      Where would you like to go next?
    </h3>
    <div className="flex flex-wrap justify-center gap-3">
      <Link to="/consultations/first-coffee" className="px-4 py-2 bg-gray-50 rounded-lg text-sm text-gray-600 hover:bg-gray-100 transition">
        ☕ Book First Coffee
      </Link>
      <Link to="/consultations/mentoring" className="px-4 py-2 bg-gray-50 rounded-lg text-sm text-gray-600 hover:bg-gray-100 transition">
        🎓 Student Mentoring
      </Link>
      <Link to="/consultations/startup" className="px-4 py-2 bg-gray-50 rounded-lg text-sm text-gray-600 hover:bg-gray-100 transition">
        🚀 Startup Advisory
      </Link>
      <Link to="/consultations/company" className="px-4 py-2 bg-gray-50 rounded-lg text-sm text-gray-600 hover:bg-gray-100 transition">
        🏢 Company Advisory
      </Link>
      <Link to="/join-us" className="px-4 py-2 bg-gray-50 rounded-lg text-sm text-gray-600 hover:bg-gray-100 transition">
        Join the Community
      </Link>
    </div>
  </div>
</section>
```

---

## Email Capture Form (Add to Homepage or Community Page)

```tsx
<section className="py-16 bg-gray-50 border-t border-gray-100">
  <div className="max-w-md mx-auto px-4 text-center">
    <h2 className="text-2xl font-bold text-gray-900 mb-3">
      Join the Robotics Community
    </h2>
    <p className="text-sm text-gray-500 mb-6">
      Monthly digest: papers, tools, internships, and challenges.
    </p>
    <form onSubmit={handleSubmit} className="flex gap-2">
      <input
        type="email"
        value={email}
        onChange={e => setEmail(e.target.value)}
        placeholder="your@email.com"
        className="flex-1 px-4 py-2.5 border border-gray-200 rounded-lg text-sm focus:outline-none focus:border-gray-400"
      />
      <button type="submit" className="px-6 py-2.5 bg-gray-900 text-white rounded-lg font-medium hover:bg-black transition">
        Join
      </button>
    </form>
  </div>
</section>
```

---

## Navigation Update (Update Existing `Navigation.tsx`)

Add "Consultations" dropdown or direct link:

```ts
// Add to navItems:
{ path: '/consultations/first-coffee', label: 'Consultations' },
```

Or better — add a dropdown with sub-items:
- First Coffee (€15)
- Student Mentoring
- Startup Advisory
- Company Advisory

---

## Routing (Add to App.tsx)

```tsx
<Route path="/consultations/first-coffee" element={<FirstCoffeePage />} />
<Route path="/consultations/mentoring" element={<MentoringPage />} />
<Route path="/consultations/startup" element={<StartupAdvisoryPage />} />
<Route path="/consultations/company" element={<CompanyAdvisoryPage />} />
```

---

## CV Submission Form Update

Add assessment questions to the existing JoinUs/CV submission flow:

```tsx
{/* After existing form fields, before submit */}
<div className="space-y-6">
  <div>
    <label className="block text-sm font-medium text-gray-900 mb-3">
      What excites you most?
    </label>
    <div className="grid grid-cols-2 gap-2">
      {['Robotics', 'AI', 'Computer Vision', 'Embedded Systems', 'Electronics', 'Mechanical Design', 'Entrepreneurship'].map(opt => (
        <label key={opt} className="flex items-center gap-2 p-3 bg-white rounded-lg border border-gray-200 cursor-pointer hover:border-gray-400">
          <input type="checkbox" value={opt} onChange={handleExcitementChange} />
          <span className="text-sm text-gray-700">{opt}</span>
        </label>
      ))}
    </div>
  </div>

  <div>
    <label className="block text-sm font-medium text-gray-900 mb-3">
      What is currently stopping you?
    </label>
    <div className="grid grid-cols-2 gap-2">
      {['No experience', 'No projects', 'No mentor', 'No confidence', 'No network', "Don't know what to learn"].map(opt => (
        <label key={opt} className="flex items-center gap-2 p-3 bg-white rounded-lg border border-gray-200 cursor-pointer hover:border-gray-400">
          <input type="checkbox" value={opt} onChange={handleBlockerChange} />
          <span className="text-sm text-gray-700">{opt}</span>
        </label>
      ))}
    </div>
  </div>

  <div>
    <label className="block text-sm font-medium text-gray-900 mb-2">
      If we met one year from today, what would success look like?
    </label>
    <textarea
      value={vision}
      onChange={e => setVision(e.target.value)}
      rows={3}
      placeholder="Describe your goals..."
      className="w-full px-4 py-2.5 border border-gray-200 rounded-lg text-sm focus:outline-none focus:border-gray-400"
    />
  </div>
</div>
```

Store these in `job_submissions` as:
- `assessmentExcitement`: array<string>
- `assessmentBlockers`: array<string>
- `assessmentVision`: string

---

## Services Page Update (Update Existing `ServicesPage.tsx`)

Replace the "Book a Consultation" section:

```tsx
{/* Remove this: */}
<p>Starting at $7 for students, $50 for companies</p>

{/* Replace with: */}
<p>Choose your path — from a €15 First Coffee to company advisory</p>
```

And update the CTA links to point to the new consultation pages instead of `/discover`.

---

## Summary of Changes

| Page | Action | Priority |
|------|--------|----------|
| `HomePage.tsx` | Update hero + 3 buttons + First Coffee teaser | High |
| `FirstCoffeePage.tsx` | New page | High |
| `MentoringPage.tsx` | New page | High |
| `StartupAdvisoryPage.tsx` | New page | Medium |
| `CompanyAdvisoryPage.tsx` | New page | Medium |
| `Navigation.tsx` | Add Consultations link | High |
| `Footer.tsx` | Add pre-footer navigation | Medium |
| `ServicesPage.tsx` | Update pricing copy | Medium |
| `JoinUs` / CV form | Add assessment questions | Medium |
| `App.tsx` | Add 4 new routes | High |
| Email capture form | Add to homepage or community page | Low |

---

## Key Principles

1. **Don't show pricing upfront** for startup/company pages — let them identify their problem first
2. **Do show pricing** for First Coffee (€15) and Mentoring (€60-120) — these are student-facing
3. **Keep Apple theme** — white, gray-50, gray-900 buttons, rounded corners, subtle borders
4. **Mobile-first** — all new pages must be responsive
5. **Bilingual** — all new pages need EN/AR support
6. **No discount framing** — remove ~~$15~~ $7 patterns, use story-based pricing
