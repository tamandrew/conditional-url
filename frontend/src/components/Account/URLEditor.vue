<script setup lang='ts'>
import { onMounted, ref, inject } from 'vue'
import type { Ref } from 'vue'
import { Conditional } from '../../types'
import ConditionalsEditor from '../ConditionalsEditor/ConditionalsEditor.vue';

const props = defineProps<{
    short: string
}>();

const conditionals: Ref<Conditional[]> = ref([]);
const error = ref("");
const accessToken: Ref<string> | undefined = inject('accessToken')

onMounted(async () => {
    if (!accessToken || !accessToken.value)
        return;

    let url;
    if (import.meta.env.PROD) {
        url = `${import.meta.env.VITE_PROD_API_URL}/api/getConditionals?short=${props.short}`;
    } else {
        url = `${import.meta.env.VITE_DEV_API_URL}/api/getConditionals?short=${props.short}`;
    }

    const response = await fetch(url, {
        method: "GET",
        headers: {
            "Content-Type": "application/json",
            "Authorization": `Bearer ${accessToken.value}`
        },
    }).then((res) => {
        if (res.status === 200) {
            return res.json();
        } else {
            return null;
        }
    });

    if (response) {
        conditionals.value = JSON.parse(response);
    }
    
})

const updateError = (msg: string) => {
    setTimeout(() => {
        error.value = "";
    }, 2000);

    error.value = msg;
}

const updateConditionalUrl = async () => {
    if (!accessToken || !accessToken.value)
        return;
        
    //verify
    for (let i = 0; i < conditionals.value.length; i++) {
        const c = conditionals.value[i];
        if (c.url == "") {
            updateError(`Please enter a URL for block #${i + 1}`);
            return;
        }
        
        if (!c.url.startsWith("http://") && !c.url.startsWith("https://")) {
            updateError(`URL #${i + 1} should start with http:// or https://`);
            return;
        }

        if (c.url.startsWith(`https://${import.meta.env.VITE_PROD_URL}`)) {
            updateError(`Can not shorten a link to this site`);
            return;
        }
        
        if (c.conditions.length == 0 && i != conditionals.value.length - 1) {
            updateError(`Please enter at least one condition for block #${i + 1}`);
            return;
        }

        for (let j = 0; j < c.conditions.length; j++) {
            const condition = c.conditions[j];
            if (condition.value === "" && condition.variable !== "URL Parameter") { //url param value can be empty
                updateError(`Please enter a value for condition #${j + 1} in block #${i + 1}`);
                return;
            } else if (condition.variable === "URL Parameter") {
                if (!condition.param) {
                    updateError(`Please enter a URL Parameter for condition #${j + 1} in block #${i + 1}`);
                    return;
                }

                if (!/^[a-zA-Z0-9]*$/.test(condition.param)) {
                    updateError(`URL Parameter for condition #${j + 1} in block #${i + 1} can only contain letters and numbers`);
                    return;
                }
            }
        }
    }

    //remove id, remove any conditions in else
    let trimmed = conditionals.value.map((c, i) => {
        return {
            url: c.url,
            and: c.and,
            conditions: i == conditionals.value.length - 1 ? [] : c.conditions
        }
    })


    let url;
    if (import.meta.env.PROD) {
        url = `${import.meta.env.VITE_PROD_API_URL}/api/updateUrl`;
    } else {
        url = `${import.meta.env.VITE_DEV_API_URL}/api/updateUrl`;
    }
    
    const response = await fetch(url, {
        method: "POST",
        headers: {
            "Content-Type": "application/json",
            "Authorization": `Bearer ${accessToken.value}`
        },
        body: JSON.stringify({
            short: props.short,
            conditionals: JSON.stringify(trimmed)
        })
    }).then((res) => {
        console.log(res)
        if (res.status === 200) {
            return res.json();
        } else if (res.status === 409) {
            updateError("URL already exists. Please enter a different short URL.")
            return null;
        } else if (res.status === 400) {
            updateError("Problem with conditionals. Please check your inputs and try again.")
            return null;
        } else {
            updateError("Something went wrong. Please try again later.")
            return null;
        }
    });

    if (response) {
        updateError("Updated successfully");
    }
}



</script>

<template>
    <div v-if = "error" class = "fixed top-4 left-4 px-4 py-1 bg-red-100 border border-black/25 rounded text-black text-center font-light">
        {{error}}
    </div>

    <div class = "w-[90%] bg-black/10 my-8 mx-auto border border-black/25 rounded-xl text-center relative">
        <div class = "mx-auto mt-4 text-white font-extralight text-xl">
            Edit Conditions
        </div>

        <ConditionalsEditor 
            :conditionals="conditionals"
            @update-conditionals="(updated) => conditionals = updated"
        />

        <button @click = "updateConditionalUrl" class = "w-full px-4 py-2 mt-6 rounded-b-xl bg-black/10 border-t border-t-black/10 text-white font-light mx-auto hover:bg-black/30 hover:text-green-100 select-none">
            Update
        </button>

    </div>
</template>
