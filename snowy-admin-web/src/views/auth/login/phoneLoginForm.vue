<template>
	<a-form ref="phoneLoginFormRef" :model="phoneFormData" :rules="formRules">
		<a-form-item name="phone">
			<a-input v-model:value="phoneFormData.phone" :placeholder="$t('login.phonePlaceholder')" size="large">
				<template #prefix>
					<mobile-outlined class="text-black text-opacity-25" />
				</template>
			</a-input>
		</a-form-item>
		<a-form-item name="phoneValidCode">
			<a-row :gutter="8">
				<a-col :span="17">
					<a-input
						v-model:value="phoneFormData.phoneValidCode"
						:placeholder="$t('login.smsCodePlaceholder')"
						size="large"
					>
						<template #prefix>
							<mail-outlined class="text-black text-opacity-25" />
						</template>
					</a-input>
				</a-col>
				<a-col :span="7">
					<a-button size="large" style="width: 100%" @click="getPhoneValidCode" :disabled="state.smsSendBtn">
						{{ (!state.smsSendBtn && $t('login.getSmsCode')) || state.time + ' s' }}
					</a-button>
				</a-col>
			</a-row>
		</a-form-item>
		<a-form-item>
			<a-button type="primary" style="width: 100%" :loading="loading" round size="large" @click="submitLogin">
				{{ $t('login.signIn') }}
			</a-button>
		</a-form-item>
	</a-form>
	<a-modal
		v-model:visible="visible"
		:width="400"
		:title="$t('login.machineValidation')"
		@cancel="handleCancel"
		@ok="handleOk"
	>
		<a-form ref="phoneLoginFormModalRef" :model="phoneFormModalData" :rules="formModalRules">
			<a-form-item name="validCode">
				<a-row :gutter="8">
					<a-col :span="17">
						<a-input
							v-model:value="phoneFormModalData.validCode"
							:placeholder="$t('login.validLaceholder')"
							size="large"
						>
							<template #prefix>
								<verified-outlined class="text-black text-opacity-25" />
							</template>
						</a-input>
					</a-col>
					<a-col :span="7">
						<img
							:src="validCodeBase64"
							style="border: 1px solid var(--border-color-split); cursor: pointer; width: 100%; height: 40px"
							@click="getPhonePicCaptcha"
						/>
					</a-col>
				</a-row>
			</a-form-item>
		</a-form>
	</a-modal>
</template>

<script setup name="smsLoginForm">
	import { message } from 'ant-design-vue'
	import { required, rules } from '@/utils/formRules'
	import loginApi from '@/api/auth/loginApi'
	import { afterLogin } from './util'

	const phoneLoginFormRef = ref()
	const phoneFormData = ref({})
	const loading = ref(false)
	let state = ref({
		time: 60,
		smsSendBtn: false
	})
	let formRules = ref({})
	const phoneValidCodeReqNo = ref('')

	// ???????????????????????????
	const getPhoneValidCode = () => {
		formRules.value.phone = [required('?????????11????????????'), rules.phone]
		delete formRules.value.phoneValidCode
		phoneLoginFormRef.value.validate().then(() => {
			// ????????????
			visible.value = true
			// ???????????????????????????
			getPhonePicCaptcha()
		})
	}
	// ??????????????????
	const submitLogin = async () => {
		formRules.value.phone = [required('?????????11????????????'), rules.phone]
		formRules.value.phoneValidCode = [required('????????????????????????'), rules.number]

		const validate = await phoneLoginFormRef.value.validate().catch(() => {})
		if (!validate) return false

		phoneFormData.value.validCode = phoneFormData.value.phoneValidCode
		// delete phoneFormData.value.phoneValidCode
		phoneFormData.value.validCodeReqNo = phoneValidCodeReqNo.value

		loading.value = true
		try {
			const token = await loginApi.loginByPhone(phoneFormData.value)
			afterLogin(token)
		} catch (err) {
			loading.value = false
		}
	}

	// ?????????
	const visible = ref(false)
	const phoneLoginFormModalRef = ref()
	const phoneFormModalData = ref({})
	const validCodeBase64 = ref('')
	const validCodeReqNo = ref('')
	const formModalRules = {
		validCode: [required('????????????????????????'), rules.lettersNum]
	}
	const getPhonePicCaptcha = () => {
		loginApi.getPicCaptcha().then((data) => {
			validCodeBase64.value = data.validCodeBase64
			phoneFormModalData.value.validCodeReqNo = data.validCodeReqNo
		})
	}
	const handleCancel = () => {
		visible.value = false
	}
	const handleOk = () => {
		// ?????????????????????????????????????????????
		phoneLoginFormModalRef.value.validate().then(() => {
			visible.value = false
			// ???????????????????????????????????????????????????
			phoneFormModalData.value.phone = phoneFormData.value.phone
			// ??????????????????????????????????????????
			state.value.smsSendBtn = true
			const interval = window.setInterval(() => {
				if (state.value.time-- <= 0) {
					state.value.time = 60
					state.value.smsSendBtn = false
					window.clearInterval(interval)
				}
			}, 1000)
			const hide = message.loading('??????????????????..', 0)

			loginApi
				.getPhoneValidCode(phoneFormModalData.value)
				.then((data) => {
					phoneValidCodeReqNo.value = data
					visible.value = false
					setTimeout(hide, 500)
					phoneFormModalData.value.validCode = ''
				})
				.catch(() => {
					setTimeout(hide, 100)
					clearInterval(interval)
					state.value.smsSendBtn = false
				})
		})
	}
</script>

<style scoped></style>
