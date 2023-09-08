https://www.notion.so/Substrate-2bfbd3a0ceeb4f39ae8d355933bc477f
果然是版本的问题
如果出错，请切换 rust 版本
rustup install nightly-2023-01-01
rustup default nightly-2023-01-01
rustup target add wasm32-unknown-unknown
cargo build --package node-template --release

```rust
// 第一行是编译标签
#![cfg_attr(not(feature = "std"), no_std)]
/// Edit this file to define custom logic or remove it if it is not needed.
/// Learn more about FRAME and the core library of Substrate FRAME pallets:
/// <https://docs.substrate.io/reference/frame-pallets/>
/// 导出poe模块的内容给其他适用
pub use pallet::*;

#[frame_support::pallet] // frame_support::pallet宏
pub mod pallet {
	// 引入frame_surpport一些预定义的依赖，包含一些常用方法、宏和接口，比如Get
	use frame_support::pallet_prelude::*;
	// 比如ensure_signed此类用于签名验证的方法
	use frame_system::pallet_prelude::*;

	/**
	 * 定义模块的配置接口
	 */
	// 添加pallet::config宏
	#[pallet::config]
	// 模块配置接口，模块接口名为Config，继承自frame_system::Config接口，
	// 就拥有该接口定义的一些数据类型，比如BlockNumber，AccountId
	pub trait Config: frame_system::Config {
		// 存证最大可接受的长度；通常情况下我们只存储链上内容的hash值，hash值的长度是固定的
		// Get<u32>: Get接口的定义的u32证书类型；需要使用Pallet::constant声明为链上常量
		#[pallet::constant]
		/// 三斜线似乎是可以在前端进行显示的内容
		type MaxClaimLength: Get<u32>;
		type RuntimeEvent: From<Event<Self>> + IsType<<Self as frame_system::Config>::RuntimeEvent>;
	}

	/**
	 * 	定义模块所需要的结构体
	 */
	// 该模块需要定义自己需要的存储项，
	// 需要使用generate_store这个宏来帮助我们生成包含所有存储项trait所有的接口
	// Blake2_128Concat是一个密码安全的哈希算法，用来对存储位置进行hash计算
	#[pallet::pallet]
	#[pallet::generate_store(pub(super) trait Store)]
	pub struct Pallet<T>(_);

	// Proofs是一个StorageMap类型
	#[pallet::storage]
	#[pallet::getter(fn proofs)]
	pub type Proofs<T: Config> = StorageMap<
		_,
		Blake2_128Concat,
		BoundedVec<u8, T::MaxClaimLength>, // u8的集合的元素类型
		/* StorageValue的value是包含两个元素的tuple，
		 * 表示哪个用户在哪个区块存储到链上的 */
		(AccountId<T>, T::BlockNumber),
	>;
	/**
	 * 定义交易的事件，可以在交易过程中进行触发
	 * 使用pallet::generate_deposit宏生成一个帮助方法，deposit_event可以用来方便调用
	 */
	#[pallet::event]
	#[pallet::generate_deposit(pub(super) fn deposit_event)]
	pub enum Event<T: Config> {
		ClaimCreated(AccountId<T>, BoundedVec<u8, T::MaxClaimLength>),
		ClaimRevoked(AccountId<T>, BoundedVec<u8, T::MaxClaimLength>),
		ClaimTransferred(AccountId<T>, AccountId<T>, BoundedVec<u8, T::MaxClaimLength>),
	}

	#[pallet::error]
	pub enum Error<T> {
		ProofAlreadyExist,
		ClaimTooLong,
		ClaimNotExist,
		NotClaimOwner,
	}

	#[pallet::hooks] // 使用这个宏来定义一些保留函数
	impl<T: Config> Hooks<BlockNumberFor<T>> for Pallet<T> {}

	/**
	 * 定义模块的可调用函数
	 */
	// 在Pallet结构体实现中添加Pallet::call宏定义可调用函数
	#[pallet::call]
	impl<T: Config> Pallet<T> {
		#[pallet::call_index(0)] // 可调用顺序
		#[pallet::weight(0)] // 指定权重
					 // origin:交易发送方；cliam：存证的内容，通常是一个hash值
		pub fn create_claim(
			origin: OriginFor<T>,
			claim: BoundedVec<u8, T::MaxClaimLength>,
		) -> DispatchResultWithPostInfo {
			// 校验发送方：是一笔签名交易
			let sender = ensure_signed(origin)?;
			// 同时验证Proofs里面不包含Bounded
			// claim这样一个键，也就是说创建时传入的claim还没有存储到proof存证存储里；
			// 如果存在就返回错误，证明这样一个proof已经被别人申请过了
			ensure!(!Proofs::<T>::contains_key(&claim), Error::<T>::ProofAlreadyExist);

			// 使用Proofs的insert方法插入键值对，键是claim，值是一个tuple，
			// 交易发送方sender及当前的区块数，区块数使用系统方法block_number获取
			Proofs::<T>::insert(
				&claim,
				(sender.clone(), frame_system::Pallet::<T>::block_number()),
			);
			// 存储成功后触发一个ClaimCreated事件，存证被创建
			Self::deposit_event(Event::ClaimCreated(sender, claim));
			Ok(().into())
		}

		#[pallet::call_index(1)]
		#[pallet::weight(0)]
		pub fn revoke_claim(
			origin: OriginFor<T>,
			claim: BoundedVec<u8, T::MaxClaimLength>,
		) -> DispatchResultWithPostInfo {
			let sender = ensure_signed(origin)?;
			// 检查存证是否存在，只有存储的存证才会被吊销；如果有，就返回存储项的值，
			// 我们上面提到了，第一个元素是凭证的所有人，第二个是区块
			let (owner, _) = Proofs::<T>::get(&claim).ok_or(Error::<T>::ClaimNotExist)?;
			// 校验发送方和owner一直
			ensure!(owner == sender, Error::<T>::NotClaimOwner);

			Proofs::<T>::remove(&claim);

			Self::deposit_event(Event::ClaimRevoked(sender, claim));
			Ok(().into())
		}

		// #[pallet::call_index(2)]
		// #[pallet::weight(0)]
		// pub fn transfer_claim(
		// 	origin: OriginFor<T>,
		// 	claim: BoundedVec<u8, T::MaxClaimLength>,
		// 	to: T::AccountId,
		// ) -> DispatchResultWithPostInfo {
		// 	let sender = ensure_signed(origin)?;
		// 	// 检查存证是否存在，只有存储的存证才会被吊销；如果有，就返回存储项的值，
		// 	// 我们上面提到了，第一个元素是凭证的所有人，第二个是区块
		// 	let (owner, block_number) =
		// 		Proofs::<T>::get(&claim).ok_or(Error::<T>::ClaimNotExist)?;
		// 	// 校验发送方和owner一直
		// 	ensure!(owner == sender, Error::<T>::NotClaimOwner);

		// 	Proofs::<T>::insert(&claim, (to.clone(), block_number));
		// 	Self::deposit_event(Event::ClaimTransferred(sender, to, claim));
		// 	Ok(().into())
		// }
	}
}

```
