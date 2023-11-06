<script setup lang="ts">
import FormViewer from '@/components/FormViewer.vue'
import {
  ref,
  watch,
  unref,
  type MaybeRef,
  reactive,
  computed,
  onMounted,
  type ComputedRef,
  type Ref,
  type UnwrapRef
} from 'vue'
import { onBeforeRouteLeave } from 'vue-router'
import { z } from 'zod'
import type { AnyZodObject, ZodFormattedError } from 'zod'

// setTimeout(() => {
//   form.name = 'jimohsodiq'
// }, 4000)
// const second_schema = reactive({
//   zod: z.object({
//     name: z.string().default('hello there'),
//     email: z.string().email()
//   }),
//   fields: {
//     name: 'an html input ref',
//     email: ' an html input ref too'
//   }
// })

function schemaDefaultValue<T extends AnyZodObject>(schema: T, key: string) {
  const typeName = schema.shape[key]._def.typeName
  // console.log(key, schema.shape[key])
  // console.log(typeName)

  // TODO: for some reason adding a default to type string with email is not tracked
  if (typeName == 'ZodDefault') {
    return schema.shape[key].safeParse(undefined).data
  }
  if (typeName == 'ZodNullable') return null
  if (typeName == 'ZodString') return ''
  if (typeName == 'ZodNumber') return 0
  if (typeName == 'ZodBoolean') return false
  // Cannot add default for ZodDate due to https://github.com/Rich-Harris/devalue/issues/51
  //if (zodType._def.typeName == "ZodDate") return new Date(NaN);
  if (typeName == 'ZodArray') return []
  if (typeName == 'ZodObject') {
    return generateObjectFromSchema(schema.shape[key])
  }
  if (typeName == 'ZodSet') return new Set()
  if (typeName == 'ZodRecord') return {}
  if (typeName == 'ZodBigInt') return BigInt(0)
  if (typeName == 'ZodSymbol') return Symbol()
  return undefined
}

type HtmlElements = HTMLInputElement | HTMLTextAreaElement | null | undefined

function schemaDefaultFields<T extends AnyZodObject>(schema: T, key: string) {
  const typeName = schema.shape[key]._def.typeName

  if (typeName == 'ZodObject') {
    return generateFieldRefsFromSchema(schema.shape[key]) as HtmlElements
  }
  return ref(null) as Ref<HtmlElements>
}

function generateFieldRefsFromSchema<T extends AnyZodObject>(schema: T): any {
  const keys = schema.keyof().Values
  const refObjects = Object.fromEntries(
    Object.keys(keys).map((key) => [key, schemaDefaultFields(schema, key) as HtmlElements] as const)
  )
  // console.log(refObjects)
  return refObjects
}

function generateObjectFromSchema<T extends AnyZodObject>(schema: T): z.infer<typeof schema> {
  type DefaultType = z.infer<typeof schema>
  const keys = schema.keyof().Values
  const defaultObject = Object.fromEntries(
    Object.keys(keys).map((key) => [key, schemaDefaultValue(schema, key)] as const)
  )
  return defaultObject as DefaultType
}

function useValidator<T extends AnyZodObject>(
  schema: MaybeRef<T>,
  option?: { validationType?: 'input' | 'touch' | 'blur' | 'none'; validateOnMount?: boolean }
) {
  const defaultOptions = {
    validationType: option?.validationType ?? 'input',
    validateOnMount: option?.validateOnMount ?? false
  }
  const tainted = ref(false)

  const unrefSchema = unref(schema)
  generateFieldRefsFromSchema(unrefSchema)

  const formData = computed(() => {
    if ('value' in schema) {
      return generateObjectFromSchema(schema.value)
    } else {
      return generateObjectFromSchema(schema)
    }
  })

  type formType = z.infer<typeof unrefSchema>
  const formFields = generateFieldRefsFromSchema(unrefSchema)

  const errorToDisplay = ref<ZodFormattedError<z.infer<typeof unrefSchema>>>(
    {} as ZodFormattedError<z.infer<typeof unrefSchema>>
  )

  // formFields[0].addEventListener('click', () => {
  //   console.log('clicked')
  // })

  // formFields[0].value.focus()

  const staticForm =
    'value' in schema ? generateObjectFromSchema(schema.value) : generateObjectFromSchema(schema)

  const form: formType = reactive(formData.value as formType)
  const result = computed(() => {
    return 'value' in schema ? schema.value.safeParse(form) : unrefSchema.safeParse(form)
  })

  async function validate() {
    const result = await unrefSchema.safeParseAsync(form)
    // console.log('valid')
    if (!result.success) {
      errorToDisplay.value = errors.value as UnwrapRef<
        ZodFormattedError<z.infer<typeof unrefSchema>>
      >
      return {
        success: false,
        error: result.error.format() as ZodFormattedError<z.infer<typeof unrefSchema>>
      }
    } else {
      errorToDisplay.value = {} as UnwrapRef<ZodFormattedError<z.infer<typeof unrefSchema>>>
      return result
    }
  }

  onMounted(async () => {
    if (defaultOptions.validateOnMount) {
      await validate()
    }
  })

  const valid = computed(() => {
    // const result = 'value' in schema ? schema.value.safeParse(form) : unrefSchema.safeParse(form)
    return result.value.success
  })

  const errors: ComputedRef<ZodFormattedError<z.infer<typeof unrefSchema>>> = computed(() => {
    // const result = unrefSchema.safeParse(form)

    if (!result.value.success) {
      return result.value.error.format() as ZodFormattedError<z.infer<typeof unrefSchema>>
    } else {
      return {} as ZodFormattedError<z.infer<typeof unrefSchema>>
    }
  })

  async function handleBlurEvent() {
    if (defaultOptions.validationType == 'blur') {
      await validate()
    }
    return
  }

  async function handleFocusEvent() {
    if (defaultOptions.validationType == 'touch') {
      await validate()
    }
    return
  }

  async function handleInputEvent() {
    if (defaultOptions.validationType === 'input') {
      await validate()
    }
    return
  }

  const binder = { onInput: handleInputEvent, onBlur: handleBlurEvent, onFocus: handleFocusEvent }

  watch(
    form,
    () => {
      if (JSON.stringify(form) == JSON.stringify(staticForm)) {
        tainted.value = false
      } else tainted.value = true
    },
    { deep: true, immediate: true }
  )

  const data = computed(() => {
    if (!result.value.success) {
      return form
    } else {
      return result.value.data
    }
  })

  return {
    form,
    valid,
    validate,
    errors: errorToDisplay,
    tainted,
    data,
    staticForm,
    binder
  }
}

const schema = z.object({
  name: z
    .string({ required_error: 'Name is required' })
    .min(7, { message: 'Must be 5 or more characters long' })
    .default('adekunle'),
  email: z.string().email('please provide a valid email address')
})
const formData = z.object({
  name: z.string().min(7).default('Ajibola'),
  contactInfo: z.object({
    email: z.string().email(),
    phone: z.string().optional().default('0123456789')
  })
})

const { form, tainted, validate, errors, binder, valid } = useValidator(formData)

// TODO: get default event if it does not pass the safeParse check
// handle errors - Done
// test with basic example from vee validate - Done
// tainted input alert when route leave - Done
// handle different validation methods e.g touch, blur, submit etc - Done
// allow for use in custom elements ( possibleSolution: for custom components, emit an onInput event and call the validate function yourself meaning: no need for binding with v-bind )

// TODO: next steps
// focus on the first element that contains error or scroll into view
// extend the functionality to allow for single zod objects
// fix bugs
// question: should schema be reactive?
// clean code
// generate key from the parent object e.g contactInfoemail and contactInfophone to avoid name collision

onBeforeRouteLeave(() => {
  if (tainted.value) {
    if (!confirm('Your changes will be lost once you leave this page')) return false
  }
})
</script>

<template>
  <div class="text-sm p-10 space-y-4">
    <div class="flex flex-col gap-1 max-w-[600px]">
      <p>Name:</p>
      <input
        class="outline-none border-[1px] border-green-400 rounded px-2 py-1"
        v-model="form.name"
        v-bind="binder"
      />
      <span v-if="errors.name?._errors[0]" class="text-red-500 text-xs font-medium">{{
        errors.name?._errors[0]
      }}</span>
    </div>
    <div class="flex flex-col gap-1 max-w-[600px]">
      <p>Contact Email:</p>

      <input
        class="outline-none border-[1px] border-green-400 rounded px-2 py-1"
        v-model="form.contactInfo.email"
        v-bind="binder"
      />
      <span v-if="errors.contactInfo?.email?._errors[0]" class="text-red-500 text-xs font-medium">{{
        errors.contactInfo?.email?._errors[0]
      }}</span>
    </div>

    <pre>Tainted: {{ tainted }}</pre>
    <pre>Valid: {{ valid }}</pre>

    <pre>Form: {{ form }}</pre>

    <pre>Errors: {{ errors }}</pre>
    <button @click="validate" class="rounded p-1.5 bg-green-500 text-green-100 font-medium">
      Validate
    </button>
    <!-- <FormViewer :error="errors" :form="form" :valid="valid" :tainted="false" /> -->
    <router-link class="block underline w-fit" to="flfll">Another page </router-link>

    <div>
      <h2 class="text-base space-y-2">How To use:</h2>
      <p class="mb-2">
        just pass in a zod schema object to useValidator with the optional parameters
      </p>
      <code>
        <pre class="">
const formData = z.object({ name: z.string().min(7).default('Ajibola'), contactInfo:
      z.object({ email: z.string().email(), phone: z.string().optional().default('0123456789') }) })
      </pre
        >
      </code>
      <code>
        <pre class="">
const { form, tainted, validate, errors, binder, valid } = useValidator(formData)</pre
        >
      </code>
    </div>
  </div>
</template>
