<script>

  let state = "start"

  let interval = null

  let range = 2



  const handler = () => {
    // var today  = new Date()
    console.log(speechSynthesis.getVoices())

    // window.speechSynthesis.

    if(state == 'stop'){
      clearInterval(interval)
      interval = null
      state = 'start'
      return
    }
    else{
      state = 'stop'
    }

    let hh = new Date().getHours()
    let mm = new Date().getMinutes()
      speechSynthesis.speak(new SpeechSynthesisUtterance(`Time is ${hh} ${mm}`));

    
    // console.log(today.getHours(),today.getMinutes());
    interval = setInterval(() => {
      let hh = new Date().getHours()
      let mm = new Date().getMinutes()
      alert(`Time is ${hh} ${mm}`)
      speechSynthesis.speak(new SpeechSynthesisUtterance(`Time is ${hh} ${mm}`));


    }, range*60*1000);


  }

</script>

  <link rel="stylesheet" href="https://unpkg.com/@picocss/pico@1.*/css/pico.min.css">



  <svelte:head>
  <title>Speaking Clock</title>

  </svelte:head>

<div data-theme='dark'  class="container">

  <article>

      <h2>Speaking clock</h2>
 
   
        <input type="range" name="range" min=2 max = 5 bind:value={range} >

          <h5>  Range is {range}
          </h5>

      <button on:click="{handler}">{state}</button>
    
    
    
    </article>
    
</div>


<style>

/* 
*{
  box-sizing: border-box;
  m;
} */




article{
  margin: 2rem auto;
  max-width: 600px;
  
  text-align: center;
}

input[type='range']{
  margin: 2rem 0 0 0;
}

div{
  /* background-color: red; */
}



</style>