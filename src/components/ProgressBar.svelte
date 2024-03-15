<script>
  export let progress = 0;

  function handleClick() {
    const targetId = 'bond'; // Replace 'target-element' with the ID of your content
    const targetElement = document.getElementById(targetId);
    if (targetElement) {
      targetElement.scrollIntoView({ behavior: 'smooth' });
    }
  }
</script>

<style>
  .progress-bar {
    position: fixed;
    top: 0;
    left: 0;
    width: 10px; /* Adjust width as needed */
    height: 100%;
    background-color: blue; /* Default color */
    cursor: pointer;
  }

  .white {
    background-color: white; /* Change color to white */
  }
</style>

<div class="progress-bar {progress === 100 ? 'white' : ''}" on:click={handleClick}></div>

