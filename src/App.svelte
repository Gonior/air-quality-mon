<script>
    import {initializeApp}from 'firebase/app'
    import {onMount} from 'svelte'
    import Speedometer from "svelte-speedometer"
    import {getAuth, signInWithEmailAndPassword} from 'firebase/auth'
    import { query, getDatabase , ref , onValue, onChildAdded} from "firebase/database"
    import Chart from 'chart.js/auto';

    
    let ppm = 0
    let Data = []
    let ctx, chartCanvas, chartCanvas2,filteredData,myChart,myChart2, averageData = [], toggleChart = false
    $: filterDate = (new Date()).toJSON().slice(0, 10);
    
    const firebaseConfig = {
            apiKey: "AIzaSyDjXMNDa5ELDImUKyPd7M3krv1CbWW5V6g",
            authDomain: "iot-air-quality-mon.firebaseapp.com",
            databaseURL: "https://iot-air-quality-mon-default-rtdb.asia-southeast1.firebasedatabase.app",
            projectId: "iot-air-quality-mon",
            storageBucket: "iot-air-quality-mon.appspot.com",
            messagingSenderId: "977563360828",
            appId: "1:977563360828:web:c2b33a05eb2afb6c7aad39",
            // @ts-ignore
            databaseURL : "https://iot-air-quality-mon-default-rtdb.asia-southeast1.firebasedatabase.app/"
        }
      
    // convert epochtime to JavaScripte Date object
    function epochToJsDate(epochTime){
        return new Date(epochTime*1000);
    }

    // convert time to human-readable format YYYY/MM/DD HH:MM:SS
    function epochToDateTime(epochTime){
        var epochDate = new Date(epochToJsDate(epochTime));
        var dateTime = epochDate.getFullYear() + "/" +
            ("00" + (epochDate.getMonth() + 1)).slice(-2) + "/" +
            ("00" + epochDate.getDate()).slice(-2) + " " +
            ("00" + epochDate.getHours()).slice(-2) + ":" +
            ("00" + epochDate.getMinutes()).slice(-2) + ":" +
            ("00" + epochDate.getSeconds()).slice(-2);

        return dateTime;
    }

    const initChart = () => {
        ctx = chartCanvas.getContext('2d');
		myChart = new Chart(ctx, {
				type: 'line',
                label : "PPM",
                data: {
                    datasets : [{
                        label : "PPM",
                        backgroundColor: 'rgb(255, 99, 132)',
                        // borderColor: 'rgb(255, 99, 132)',
                        data : [],
                    }]
                },
                options: {
                    scales: {
                        // @ts-ignore
                        xAxis: [{
                            type: "time",
                            distribution: "linear",
                                time: {
                                    unit: 'month'
                                }
                        }]
                }
            }
		});
    }
    const initBarChart = () => {
        ctx = chartCanvas2.getContext('2d');
		myChart2 = new Chart(ctx, {
            type: 'bar',
            // @ts-ignore
            label : "PPM",
            data: {
                datasets : [{
                    label : "PPM",
                    backgroundColor: 'rgb(255, 99, 132)',
                    // borderColor: 'rgb(255, 99, 132)',
                    data : [],
                }]
            },
		});
    }

    onMount( async () => {
        initChart()
        initBarChart()
        const app = initializeApp(firebaseConfig)
        const database = getDatabase(app) 
        const auth = getAuth()
        let signIn = await signInWithEmailAndPassword(auth, "iairquality67@gmail.com","unikom2018")
        let uid = signIn ? signIn.user.uid : null 
        const ppmRef = ref(database, `UsersData/${uid}/readings`)
        
        await onValue(ppmRef, (snapshot) => {
            console.log('get data once')
            const json = snapshot.toJSON()
            let arr = Object.keys(json).map(function(key) {
                return json[key];
            });
            
            arr.forEach(e => {
                // e => { "ppm" : 2.4 , "timestamp" : 1660742888339 , hours : 10}
                let obj = {}
                obj = {
                    y : e.ppm,
                    x : epochToDateTime(e.timestamp),
                    hours : epochToJsDate(e.timestamp).getHours()
                }
                
                Data = [...Data, obj]
            });

            for (let i = 0; i <= 23; i++) {
                let avgPpm = Data.filter(d => d.hours === i)
                let sum = 0
                avgPpm.forEach(item => {
                    sum += parseFloat(item.y)
                })
                let dataPerHours = {
                    x : i < 10 ? `0${i}:00` : `${i}:00`,
                    // @ts-ignore
                    y : avgPpm.length === 0 ? 0 : parseFloat(sum/avgPpm.length).toFixed(2)
                }   

                
                averageData = [...averageData, dataPerHours]
            }
            
        }, {onlyOnce : true})
        setTimeout(() => {
            onValue(ppmRef, (snapshot) => {
                console.log('get data continues')
                const json = snapshot.toJSON()
                let arr = Object.keys(json).map(function(key) {
                    return json[key];
                });
                let obj = {}
                arr.forEach(e => {
                    // e => { "ppm" : 2.4 , "timestamp" : 1660742888339 , hours : 10}
                    obj = {
                        y : e.ppm,
                        x : epochToDateTime(e.timestamp),
                        hours : epochToJsDate(e.timestamp).getHours()
                    }
                    
                    
                });
                Data.push(obj)
                Data = [...Data]
                let dataPerHours
                for (let i = 0; i <= 23; i++) {
                    let avgPpm = Data.filter(d => d.hours === i)
                    let sum = 0
                    avgPpm.forEach(item => {
                        sum += parseFloat(item.y)
                    })
                    dataPerHours = {
                        x : i < 10 ? `0${i}:00` : `${i}:00`,
                        // @ts-ignore
                        y : avgPpm.length === 0 ? 0 : parseFloat(sum/avgPpm.length).toFixed(2)
                    }   

                    
                    
                }
                averageData.push(dataPerHours)
                averageData = [...averageData]
                
                filteredData = [...Data.filter((d) => d.x.slice(0,10).replaceAll("/","-") === filterDate)]
                myChart.data.datasets.forEach((dataset) => {
                    dataset.data = [...filteredData];
                });
                myChart.update()

                myChart2.data.datasets.forEach((dataset) => {
                    dataset.data = [...averageData];
                });
                myChart2.update()
                console.log(averageData)
                console.log(filteredData)
            }, (errorObject) => {
                console.log('The read failed: ' + errorObject);
            })
        },0)
        


        onChildAdded(ppmRef, (data) => {
            // @ts-ignore
            ppm = data.toJSON().ppm
        })

    })

    const pickADate = (e) => {

        filteredData = [...Data.filter((d) => d.x.slice(0,10).replaceAll("/","-") === e.target.value)]
        myChart.data.datasets.forEach((dataset) => {
        dataset.data = [...filteredData];
        });
        myChart.update()
    }
    
    let aqi = [
        {
            level : "25",
            effect : "PEL (Long Term)",
            duration : "8 hrs"
        },
        {
            level : "200",
            effect : "Slight headache, discomport",
            duration : "3 hrs"
        },
        {
            level : "400",
            effect : "Headache, discomport",
            duration : "2 hrs"
        },
        {
            level : "600",
            effect : "Headache, discomport",
            duration : "1 hrs"
        },
        {
            level : "600 - 1000",
            effect : "Confusion, headache, nause",
            duration : "1 - 2 hrs"
        },
        {
            level : "1000 - 2000",
            effect : "Tedency to stagger",
            duration : "30 mins"
        },
        {
            level :"2000 - 2500",
            effect : "Unconsciousness",
            duration : "30 mins"
        },
        {
            level : "4000",
            effect : "Fatal",
            duration : "< 1 hr"
        },

    ]
const toggleChartFunc = () => {
    toggleChart = !toggleChart
}
</script>

<div class="h-screen flex flex-col p-4">
    <div class="flex-0 flex">
        <h1 class="font-semibold">Air Quality Monitoring System</h1>
    </div>
    <div class="flex-1 overflow-y-auto my-5 w-full px-4">
        <div class="w-full flex">
            <div class="w-1/2 flex flex-col items-center justify-center">
                <h1 class="mb-4">The Latest PPM Value</h1>
                <Speedometer
                    
                    value={ppm > 4000 ? 4000 : ppm}
                    currentValueText ="{ppm} PPM"
                    maxValue={4000}
                    segments={8}
                    
                    segmentColors={['#37F72B', '#EFE920', '#F69724',"#FB2D2D","#AE23FE","#c0c0c0","#525E75","#000"]}
                
                    needleColor="steelblue"
                    needleTransitionDuration={1000}
                    needleTransition="easeQuadInOut"
                />
            </div>
            <div class="w-1/2">
                <div class="flex justify-between">
                    {#if !toggleChart}
                        <p class="font-semibold">Data PPM per Day</p>
                        <input type="date" value={filterDate} on:input={e => 
                        // @ts-ignore
                        pickADate(e) || filterDate}/>
                    {:else}
                        <p class="font-semibold">The Highest PPM by hours</p>
                    {/if}
                </div>  
                
                <canvas bind:this={chartCanvas} style="display: {!toggleChart ? 'block' : 'none'};"></canvas>
        
                <canvas bind:this={chartCanvas2}  style="display: {!toggleChart ? 'none' : 'block'};"></canvas>
                <div class="flex justify-end mt-5 ">
                    <button class="rounded-lg bg-blue-300 text-white hover:bg-blue-400 px-4 py-2 font-semibold" on:click={() => toggleChartFunc()}>
                        {!toggleChart ? "Next Chart" : "Prev Chart"}
                        <!-- <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" class="h-5 w-5"><path style="fill:none;stroke:currentColor;stroke-linecap:round;stroke-linejoin:round;stroke-width:2" d="M18 3 6 12l12 9"/></svg> -->
                    </button>
                    <!-- <button class="text-gray-400 hover:text-gray-600">
                        <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" class="h-5 w-5"><path style="fill:none;stroke:currentColor;stroke-linecap:round;stroke-linejoin:round;stroke-width:2" d="m7.5 3 9 9-9 9"/></svg>
                    </button> -->
                </div>            
            </div>
        </div>
        <div class="flex items-center  flex-col justify-center mt-20">
            <p class="">Index Air Quality Carbon Monoxide (CO) <a href="https://petrotrainingasia.com/bekerja-di-ruang-terbatas/" class="text-blue-500">Sumber</a></p>
            
            <table class="w-1/2">
                <thead class="bg-white border-b">
                  <tr>
                    <th scope="col" class="text-sm font-medium text-gray-900 px-6 py-4 text-center">
                      Level (PPM)
                    </th>
                    <th scope="col" class="text-sm font-medium text-gray-900 px-6 py-4 text-center">
                      Effects Sysmptoms
                    </th>
                    <th scope="col" class="text-sm font-medium text-gray-900 px-6 py-4 text-center">
                      Duration
                    </th>
                  </tr>
                </thead>
                <tbody>
                {#each aqi as a}    
                
                <tr class="bg-white border-b transition duration-300 ease-in-out hover:bg-gray-100">
                    <td class="text-sm text-center font-bold text-gray-900  px-6 py-4 whitespace-nowrap">
                        {a.level}
                    </td>
                    <td class="text-sm text-gray-900 font-light px-6 py-4 whitespace-nowrap">
                        {a.effect}
                    </td>
                    <td class="text-sm text-center text-gray-900 font-light px-6 py-4 whitespace-nowrap">
                        {a.duration}
                    </td>
                </tr>
                {/each}
                  
                </tbody>
            </table>
        </div>
        
    </div>
    <div class="flex-0 text-center">
        <p>powered by Svelte, Tailwind, Vite, & Firebase</p>
    </div>
</div>