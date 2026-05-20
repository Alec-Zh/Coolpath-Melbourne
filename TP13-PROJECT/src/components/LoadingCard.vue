<script setup>
defineProps({
  title: {
    type: String,
    default: 'Loading...',
  },
  subtitle: {
    type: String,
    default: '',
  },
})
</script>

<template>
  <section class="loading-plan-section">
    <div class="loading-plan-card">
      <div class="loading-spinner">
        <svg width="48" height="48" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round">
          <circle cx="12" cy="12" r="10" stroke-opacity="0.25" />
          <path d="M12 2a10 10 0 0 1 10 10" />
        </svg>
      </div>
      <h3>{{ title }}</h3>
      <p v-if="subtitle">{{ subtitle }}</p>
    </div>
  </section>
</template>

<style scoped>
.loading-plan-section {
  display: flex;
  justify-content: center;
  padding: 3rem 1rem;
}

.loading-plan-card {
  background: linear-gradient(135deg, #e8f5e9, #e3f2fd);
  border: 2px solid #c8e6c9;
  border-radius: 20px;
  padding: 2.5rem 2rem;
  text-align: center;
  max-width: 400px;
  width: 100%;
}

.loading-spinner {
  color: #2d7a3a;
  margin-bottom: 1.25rem;
  animation: spin 1.2s linear infinite;
}

.loading-plan-card h3 {
  font-family: 'Fraunces', 'Georgia', serif;
  font-size: 1.4rem;
  font-weight: 700;
  color: #1c2e2a;
  margin: 0 0 0.5rem;
}

.loading-plan-card p {
  font-size: 1.02rem;
  color: #5a6e6a;
  margin: 0;
}

@keyframes spin {
  from { transform: rotate(0deg); }
  to { transform: rotate(360deg); }
}
</style>
