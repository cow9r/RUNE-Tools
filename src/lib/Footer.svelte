<script>
  import { onMount, onDestroy } from 'svelte';
  import { fade, slide } from 'svelte/transition';
  import { cubicInOut } from 'svelte/easing';
  import { audioPlaying } from './stores/audioStore';

  const emojis = ['🫡', '✍️', '💪', '🧙‍♂️', '🕺', '🏃‍♂️‍➡️', '🦅', '🐋', '🐉', '⚡️', '🌊', '🍷', '🍻', '🏄‍♂️', '🏆', '🎸', '🚀', '🗿', '🗽', '🏗️', '📠', '🔌', '🔮', '🔭', '💯', '🏴‍☠️', '🥷', '👑', '🪐', '🍦', '🍾', '🎯', '❤️', '☑️', '🆒'];

  function getRandomEmoji() {
    return emojis[Math.floor(Math.random() * emojis.length)];
  }

  let randomEmoji;
  let currentPage = 0;
  let startX = 0;
  let startY = 0;
  let isDragging = false;
  const totalPages = 2;
  let autoScrollTimer;
  let isUserInteracting = false;

  const FIRST_PAGE_DURATION = 15000;  // 15 seconds
  const OTHER_PAGE_DURATION = 10000;  // 10 seconds

  // Music tracks array
  const musicTracks = [
    '/assets/music/Also-Sprach-Zarathustra-PM-Music.mp3',
    '/assets/music/Bach-Cello-Suite-No.-1-G-Major-PM-Music.mp3',
    '/assets/music/Blue-Danube-PM-Music.mp3',
    '/assets/music/Ride-of-the-Valkyries-PM-Music.mp3',
    '/assets/music/Romeo-and-Juliet-PM-Music.mp3',
    '/assets/music/Russian-Dance-PM-Music.mp3',
    '/assets/music/The-Flower-Duet-PM-Music.mp3',
    '/assets/music/We-Shop-Song-PM-Music.mp3',
    '/assets/music/Winter-Vivaldi-PM-Music.mp3'
  ];

  let audio;
  let currentTrackIndex = 0;
  let currentTrackTitle = '';

  function getTrackTitle(path) {
    // Extract title from path and format it
    return path
      .split('/')
      .pop()
      .replace('-PM-Music.mp3', '')
      .replace(/-/g, ' ');
  }

  const pages = [
    {
      content: {
        type: 'links',
        elements: [
          { href: "https://github.com/cow9r/RUNE-Tools", text: "Source" },
          { text: " by " },
          { href: "https://x.com/familiarcow", text: "familiarcow" },
          { emoji: true },
          { href: "https://x.com/RuneDotTools", text: "Follow on 𝕏" },
          { type: 'sound' } // Add sound button element
        ]
      }
    },
    {
      content: {
        type: 'vultisig',
        href: "https://t.me/vultirefbot/app?startapp=ref_3a5c3bba-9c5f-47ed-a2fc-6f659476404a",
        text: "Secure your RUNE & register for the Vultisig airdrop campaign"
      }
    }
  ];

  function startAutoScroll() {
    clearTimeout(autoScrollTimer);
    const duration = currentPage === 0 ? FIRST_PAGE_DURATION : OTHER_PAGE_DURATION;
    
    autoScrollTimer = setTimeout(() => {
      if (!isUserInteracting) {
        if (currentPage < totalPages - 1) {
          currentPage++;
          startAutoScroll(); // Schedule next scroll
        } else {
          currentPage = 0; // Reset to first page
          startAutoScroll(); // Restart cycle
        }
      }
    }, duration);
  }

  function handleUserInteraction() {
    isUserInteracting = true;
    clearTimeout(autoScrollTimer);
    
    // Reset auto-scroll after 30 seconds of no interaction
    setTimeout(() => {
      isUserInteracting = false;
      startAutoScroll();
    }, 30000);
  }

  function handleTouchStart(e) {
    startX = e.touches[0].clientX;
    startY = e.touches[0].clientY;
    isDragging = true;
    handleUserInteraction();
  }

  function handleTouchMove(e) {
    if (!isDragging) return;
    
    const currentX = e.touches[0].clientX;
    const currentY = e.touches[0].clientY;
    const diffX = startX - currentX;
    const diffY = startY - currentY;
    
    // If horizontal movement is greater than vertical, handle as horizontal swipe
    if (Math.abs(diffX) > Math.abs(diffY)) {
      if (Math.abs(diffX) > 20) {
        if (diffX > 0) {
          // Swipe left - next page
          currentPage = (currentPage + 1) % totalPages;
          isDragging = false;
        } else {
          // Swipe right - previous page
          currentPage = currentPage === 0 ? totalPages - 1 : currentPage - 1;
          isDragging = false;
        }
      }
    } else if (Math.abs(diffY) > 20) {
      // Handle vertical swipe as before
      if (diffY > 0) {
        currentPage = (currentPage + 1) % totalPages;
        isDragging = false;
      } else if (diffY < 0) {
        currentPage = currentPage === 0 ? totalPages - 1 : currentPage - 1;
        isDragging = false;
      }
    }
  }

  function handleTouchEnd() {
    isDragging = false;
  }

  function handleWheel(e) {
    handleUserInteraction();
    if (Math.abs(e.deltaY) > 10) {
      if (e.deltaY > 0) {
        currentPage = (currentPage + 1) % totalPages;
      } else if (e.deltaY < 0) {
        currentPage = currentPage === 0 ? totalPages - 1 : currentPage - 1;
      }
    }
  }

  // Add keyboard navigation
  function handleKeydown(e) {
    if (e.key === 'ArrowUp' || e.key === 'ArrowDown') {
      handleUserInteraction();
      if (e.key === 'ArrowDown') {
        currentPage = (currentPage + 1) % totalPages;
      } else {
        currentPage = currentPage === 0 ? totalPages - 1 : currentPage - 1;
      }
    }
  }

  function toggleSound() {
    if (!audio) {
      // Initialize audio on first play
      audio = new Audio();
      audio.addEventListener('ended', playNextTrack);
      currentTrackIndex = Math.floor(Math.random() * musicTracks.length);
      audio.src = musicTracks[currentTrackIndex];
      currentTrackTitle = getTrackTitle(musicTracks[currentTrackIndex]);
    }

    if ($audioPlaying) {
      audio.pause();
      audioPlaying.set(false);
    } else {
      audio.play().catch(err => {
        console.error('Error playing audio:', err);
        audioPlaying.set(false);
      });
      audioPlaying.set(true);
    }
  }

  function playNextTrack() {
    if (!$audioPlaying) return;
    
    currentTrackIndex = (currentTrackIndex + 1) % musicTracks.length;
    audio.src = musicTracks[currentTrackIndex];
    currentTrackTitle = getTrackTitle(musicTracks[currentTrackIndex]);
    audio.play().catch(err => {
      console.error('Error playing next track:', err);
      audioPlaying.set(false);
    });
  }

  onMount(() => {
    randomEmoji = getRandomEmoji();
    startAutoScroll();
  });

  onDestroy(() => {
    clearTimeout(autoScrollTimer);
    if (audio) {
      audio.removeEventListener('ended', playNextTrack);
      audio.pause();
      audio = null;
    }
  });
</script>

<footer
  on:touchstart={handleTouchStart}
  on:touchmove={handleTouchMove}
  on:touchend={handleTouchEnd}
  on:wheel={handleWheel}
  on:mouseenter={handleUserInteraction}
  on:keydown={handleKeydown}
>
  <div 
    class="page-container"
    on:click={() => {
      currentPage = (currentPage + 1) % totalPages;
      handleUserInteraction();
    }}
  >
    {#key currentPage}
      <div 
        class="page"
        in:slide={{ 
          duration: 300,
          easing: cubicInOut,
          axis: 'y'
        }}
      >
        {#if pages[currentPage].content.type === 'links'}
          <span>
            {#each pages[currentPage].content.elements as element}
              {#if element.type === 'sound'}
                <button
                  class="sound-button"
                  on:click|stopPropagation={toggleSound}
                  aria-label={$audioPlaying ? 'Stop music' : 'Play music'}
                  title={currentTrackTitle}
                >
                  {#if $audioPlaying}
                    🔊
                  {:else}
                    🔈
                  {/if}
                </button>
              {:else if element.href}
                <a 
                  href={element.href} 
                  target="_blank" 
                  class="source-link"
                  on:click|stopPropagation
                >
                  {element.text}
                </a>
              {:else if element.emoji}
                {" "}{getRandomEmoji()}{" "}
              {:else}
                {element.text}
              {/if}
            {/each}
          </span>
        {:else if pages[currentPage].content.type === 'vultisig'}
          <span>
            <a 
              href={pages[currentPage].content.href}
              target="_blank" 
              class="source-link"
              on:click|stopPropagation
            >
              {pages[currentPage].content.text}
            </a>
          </span>
        {:else if pages[currentPage].content.type === 'feedback'}
          <span>
            <a 
              href={pages[currentPage].content.href}
              target="_blank" 
              class="source-link"
              on:click|stopPropagation
            >
              {pages[currentPage].content.text}
            </a>
          </span>
        {/if}
      </div>
    {/key}
  </div>
</footer>

<style>
  footer {
    padding: 0.5rem 1.5rem;
    background-color: var(--surface-color);
    box-shadow: 0 -1px 0 rgba(255, 255, 255, 0.1);
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 0.25rem;
    user-select: none;
    touch-action: pan-x pan-y;
    position: fixed;
    bottom: 0;
    left: 0;
    right: 0;
    min-height: 32px;
    z-index: 100;
    border-top: 1px solid rgba(255, 255, 255, 0.05);
    background: linear-gradient(
      to bottom,
      rgba(44, 44, 44, 0.95),
      var(--surface-color)
    );
  }

  .page-container {
    position: relative;
    width: 100%;
    display: flex;
    justify-content: center;
    overflow: hidden;
    height: 24px;
    margin-top: 0;
    cursor: pointer;
  }

  .page {
    width: 100%;
    text-align: center;
    font-size: 0.95rem;
    font-weight: 400;
    line-height: 24px;
    white-space: nowrap;
    overflow: hidden;
    color: rgba(255, 255, 255, 0.8);
    padding-right: 1.5rem;
  }

  /* Add media query for smaller screens */
  @media (max-width: 600px) {
    .page {
      font-size: 0.8rem;
      padding-right: 1rem;
    }
    
    /* Adjust container height for smaller text */
    .page-container {
      height: 20px;
    }
    
    /* Adjust line height to match new container height */
    .page {
      line-height: 20px;
    }
    
    footer {
      padding: 0.25rem 1rem;
    }
  }

  @media (max-width: 400px) {
    .page {
      font-size: 0.75rem;
      padding-right: 0.75rem;
    }
  }

  /* Reset all link styles in footer */
  footer a,
  footer a:visited,
  footer a:active,
  footer a:link {
    color: #31FD9D;
    text-decoration: none;
  }

  .source-link,
  .source-link:visited,
  .source-link:active,
  .source-link:link {
    color: #31FD9D;
    text-decoration: none;
    font-weight: 600;
    transition: opacity 0.2s cubic-bezier(0.4, 0, 0.2, 1);
    opacity: 0.95;
  }

  .source-link:hover {
    opacity: 1;
  }

  footer span {
    color: rgba(255, 255, 255, 0.8);
  }

  /* Add styles for sound button */
  .sound-button {
    background: none;
    border: none;
    cursor: pointer;
    font-size: 0.95rem;
    padding: 0 0.5rem;
    color: rgba(255, 255, 255, 0.8);
    transition: opacity 0.2s ease;
    vertical-align: middle;
    line-height: inherit;
  }

  .sound-button:hover {
    opacity: 0.7;
  }

  /* Add media queries for the sound button to match text sizes */
  @media (max-width: 600px) {
    .sound-button {
      font-size: 0.8rem;
    }
  }

  @media (max-width: 400px) {
    .sound-button {
      font-size: 0.75rem;
    }
  }
</style> 