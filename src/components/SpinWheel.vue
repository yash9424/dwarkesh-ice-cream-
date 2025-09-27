<template>
  <div ref="container" class="flex spin-container">
    <picture>
      <source srcset="/img/image.avif" type="image/avif" />
      <source srcset="/img/image.webp" type="image/webp" />
      <img src="/img/image.png" class="image" alt="background image" />
    </picture>
    <div
      class="icon"
      @click="spin"
      @keyup.enter="spin"
      @keyup.space="spin"
      v-tooltip.bottom="{
        value: `↻ Spin!`,
        class: 'text-xl',
        escape: true
      }"
      tabindex="0"
    ></div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, inject, watch } from 'vue';
import random from 'random';
import { Wheel, type WheelProps } from 'spin-wheel';
import { useDialog } from 'primevue/usedialog';
import { TickSound, LabelLength } from '@/services/SettingService';
import { GroupLabel, GroupLabels, ItemService, Items } from '@/services/ItemService';
import CongratulationDialog from '@/components/CongratulationDialog.vue';

const itemService = inject<ItemService>('ItemService');

const properties: WheelProps = {
  // debug: import.meta.env.DEV,
  isInteractive: false,
  radius: 0.48,
  rotationResistance: 0,
  itemLabelRadius: 0.92,
  itemLabelRadiusMax: 0.3,
  itemLabelRotation: 180,
  itemLabelAlign: 'left',
  itemLabelColors: ['#fff'],
  itemLabelBaselineOffset: -0.07,
  // Should also change app.scss
  itemLabelFont:
    '"Suez One", "Mochiy Pop P One", "Jua", "Unbounded", "Mitr", "Noto Sans TC", "Noto Sans SC", "Noto Sans Lao", "Noto Color Emoji"',
  itemLabelFontSizeMax: 55,
  itemBackgroundColors: [
    '#fdc963',
    '#00cca8',
    '#2b87e9',
    '#fd775b',
    '#ff4b78',
    '#c88857',
    '#a64a97',
    '#5b7c7d',
    '#715344',
    '#904e55',
    '#8b7856'
  ],
  rotationSpeedMax: 2000,
  lineWidth: 1,
  lineColor: '#fff',
  items: []
};

const container = ref();

let spinCount = 0;
let wheel: Wheel | undefined = undefined;

const stopAndClearSound = () => {
  if (!wheel) return;

  wheel.onCurrentIndexChange = () => {};
  wheel.stop();
};

const playSound = () => {
  if (!TickSound.value) return;

  var src = TickSound.value.value.startsWith('data:')
    ? TickSound.value.value
    : `/sound/${TickSound.value.value}`;
  const audio = new Audio(src);
  audio.volume = 0.3;
  audio.play();
};

const spin = () => {
  if (!wheel) return;

  const targetIndex = getWeightedRandomIndex();
  const targetAngle = (360 / fixedItems.length) * targetIndex + (360 / fixedItems.length) / 2;
  const spins = 5 + Math.random() * 3;
  const finalAngle = spins * 360 + targetAngle;

  wheel.onCurrentIndexChange = () => {
    if (!wheel) return;
    playSound();

    switch (true) {
      case wheel.rotationSpeed < 400:
        wheel.rotationResistance = -100;
        break;
      case wheel.rotationSpeed < 100:
        wheel.rotationResistance = -30;
        break;
      case wheel.rotationSpeed < 30:
        wheel.rotationResistance = -10;
        break;
    }
  };

  wheel.rotationResistance = -400;
  wheel.spinToItem(targetIndex, 2000 + Math.random() * 1000);
};

const dialog = useDialog();
const openCongratulationDialog = ($event: {
  type: 'rest';
  currentIndex: number;
  rotation: number;
}) => {
  const item = fixedItems[$event.currentIndex];
  const label = item.label;
  
  const isWinner = label.includes('Asli Aam') || label.includes('Mangodoli') || label.includes('Chocobar');
  const isBadLuck = label.includes('Better luck next time');
  
  if (isWinner) {
    createConfetti();
    container.value.classList.add('winner-glow');
    setTimeout(() => container.value.classList.remove('winner-glow'), 3000);
  } else if (isBadLuck) {
    container.value.classList.add('shake');
    setTimeout(() => container.value.classList.remove('shake'), 1000);
  }
  
  dialog.open(CongratulationDialog, {
    props: {
      modal: true,
      showHeader: false,
      style: 'border: 0',
      contentStyle: 'border: 0; backgroundColor: transparent',
      dismissableMask: true
    },
    data: {
      item: item,
      isWinner: isWinner,
      isBadLuck: isBadLuck
    }
  });
};

const createConfetti = () => {
  const colors = ['#FFD700', '#FF6B6B', '#4ECDC4', '#45B7D1', '#96CEB4', '#FECA57', '#FF9FF3'];
  const shapes = ['🎉', '🎊', '⭐', '✨', '🌟', '💫'];
  const confettiContainer = document.createElement('div');
  confettiContainer.style.position = 'fixed';
  confettiContainer.style.top = '0';
  confettiContainer.style.left = '0';
  confettiContainer.style.width = '100%';
  confettiContainer.style.height = '100%';
  confettiContainer.style.pointerEvents = 'none';
  confettiContainer.style.zIndex = '9999';
  document.body.appendChild(confettiContainer);
  
  for (let i = 0; i < 80; i++) {
    const confetti = document.createElement('div');
    const isEmoji = Math.random() > 0.6;
    
    confetti.style.position = 'absolute';
    confetti.style.left = Math.random() * 100 + '%';
    confetti.style.top = '-20px';
    
    if (isEmoji) {
      confetti.textContent = shapes[Math.floor(Math.random() * shapes.length)];
      confetti.style.fontSize = (12 + Math.random() * 8) + 'px';
    } else {
      confetti.style.width = (8 + Math.random() * 6) + 'px';
      confetti.style.height = (8 + Math.random() * 6) + 'px';
      confetti.style.backgroundColor = colors[Math.floor(Math.random() * colors.length)];
      confetti.style.borderRadius = Math.random() > 0.5 ? '50%' : '0';
    }
    
    const duration = 3 + Math.random() * 3;
    const delay = Math.random() * 1.5;
    confetti.style.animation = `confetti-fall ${duration}s ${delay}s ease-out forwards`;
    confettiContainer.appendChild(confetti);
  }
  
  setTimeout(() => {
    if (document.body.contains(confettiContainer)) {
      document.body.removeChild(confettiContainer);
    }
  }, 7000);
};

const fixedItems = [
  { label: 'Better luck next time ❌', weight: 1 },
  { label: 'Asli Aam 🥭', weight: 1 },
  { label: 'Mangodoli 🍦', weight: 1 },
  { label: 'Chocobar 🍫', weight: 1 },
  { label: 'Choco Brownie 🧁', weight: 1 },
  { label: '₹1000 💰', weight: 1 }
];

const actualWeights = [5, 1, 2, 2, 0, 0];

const getWeightedRandomIndex = () => {
  const totalWeight = actualWeights.reduce((sum, weight) => sum + weight, 0);
  let randomNum = Math.random() * totalWeight;
  
  for (let i = 0; i < actualWeights.length; i++) {
    randomNum -= actualWeights[i];
    if (randomNum <= 0) {
      return i;
    }
  }
  return 0;
};

onMounted(() => {
  watch(LabelLength, (newValue) => {
    wheel!.itemLabelRadiusMax = 1 - newValue;
  });

  wheel = new Wheel(container.value, {
    ...properties,
    items: fixedItems,
    itemLabelRadiusMax: 1 - LabelLength.value
  });

  wheel.spin(10);

  wheel.onRest = ($event) => {
    stopAndClearSound;
    openCongratulationDialog($event);
  };

  wheel.onSpin = () => {
    gtag('event', 'spin');
    gtag('event', 'spin_count', {
      count: ++spinCount
    });
  };

  // Workaround for itemLabelRadiusMax not working on first load.
  setTimeout(() => {
    wheel!.itemLabelRadiusMax = 1 - LabelLength.value;
  }, 50);
});
</script>

<style lang="scss" scoped>
@import 'primeflex/core/_variables.scss';

.spin-container {
  aspect-ratio: 1/1;
  width: 200vw;
  height: 90vh;

  margin-top: -3.5rem;
  margin-bottom: -10vh;
  position: relative;

  @media (min-width: map-get($breakpoints, 'sm')) {
    height: 100vh;
  }

  @media (min-width: map-get($breakpoints, 'md')) {
    height: 110vh;
  }
}

.image {
  object-position: center;
  object-fit: contain;

  aspect-ratio: 1/1;
  width: 200vw;
  height: 90vh;

  position: absolute;
  top: calc(calc(50%) - calc(90vh / 2));
  left: calc(calc(50%) - calc(200vw / 2));

  @media (min-width: map-get($breakpoints, 'sm')) {
    height: 100vh;
    top: calc(calc(50%) - calc(100vh / 2));
  }

  @media (min-width: map-get($breakpoints, 'md')) {
    height: 110vh;
    top: calc(calc(50%) - calc(110vh / 2));
  }
}

.button-container {
  margin-top: -5.5rem;

  button {
    z-index: 2;
    position: relative;

    $background-color: #0c0f1d;
    background: $background-color;

    &:hover {
      filter: brightness(1.3);
    }
  }
}

.icon {
  $icon-size: 13vh;
  cursor: pointer;

  width: $icon-size;
  height: $icon-size;
  border-radius: 50%;

  background-image: url(/img/icon.png);
  background-image: -webkit-image-set(
    url(/img/icon.avif) type('image/avif'),
    url(/img/icon.webp) type('image/webp'),
    url(/img/icon.png) type('image/png')
  );
  background-image: image-set(
    url(/img/icon.avif) type('image/avif'),
    url(/img/icon.webp) type('image/webp'),
    url(/img/icon.png) type('image/png')
  );

  background-size: contain;

  position: absolute;
  top: calc(calc(50%) - calc($icon-size / 2));
  left: calc(calc(50%) - calc($icon-size / 2));

  &:hover {
    filter: brightness(1.1);
  }
}

.winner-glow {
  animation: winner-glow 3s ease-in-out, winner-pulse 0.5s ease-in-out;
  box-shadow: 0 0 40px #ffd700, 0 0 80px #ffd700, 0 0 120px #ffd700;
}

.shake {
  animation: shake-intense 1s ease-in-out;
}

@keyframes winner-glow {
  0%, 100% { 
    box-shadow: 0 0 20px #ffd700, 0 0 40px #ffd700;
    filter: brightness(1);
  }
  50% { 
    box-shadow: 0 0 60px #ffd700, 0 0 120px #ffd700, 0 0 180px #ffd700;
    filter: brightness(1.3);
  }
}

@keyframes winner-pulse {
  0% { transform: scale(1); }
  50% { transform: scale(1.05); }
  100% { transform: scale(1); }
}

@keyframes shake-intense {
  0%, 100% { transform: translateX(0) rotate(0deg); }
  10% { transform: translateX(-15px) rotate(-2deg); }
  20% { transform: translateX(15px) rotate(2deg); }
  30% { transform: translateX(-12px) rotate(-1deg); }
  40% { transform: translateX(12px) rotate(1deg); }
  50% { transform: translateX(-8px) rotate(-0.5deg); }
  60% { transform: translateX(8px) rotate(0.5deg); }
  70% { transform: translateX(-4px) rotate(-0.2deg); }
  80% { transform: translateX(4px) rotate(0.2deg); }
  90% { transform: translateX(-2px) rotate(-0.1deg); }
}

@keyframes confetti-fall {
  0% {
    transform: translateY(-100vh) rotate(0deg) scale(1);
    opacity: 1;
  }
  10% {
    opacity: 1;
  }
  90% {
    opacity: 0.8;
  }
  100% {
    transform: translateY(100vh) rotate(720deg) scale(0.5);
    opacity: 0;
  }
}
</style>
