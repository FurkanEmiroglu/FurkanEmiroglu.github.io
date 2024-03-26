## Generic Object Pooling



```csharp
using System.Collections.Generic;
using UnityEngine;

namespace FE.ObjectPool
{
    public abstract class PoolBase<T> : MonoBehaviour where T : MonoBehaviour
    {
        /// <summary>
        /// Prefab to be pooled, will use it as a template to instantiate new items
        /// </summary>
        [SerializeField] 
        private T _objectPrefab;
        
        /// <summary>
        /// Pool size
        /// </summary>
        [SerializeField] 
        private int _initialCount;

        /// <summary>
        /// pooled items
        /// </summary>
        private readonly Queue<T> m_poolQueue = new();

        /// <summary>
        /// Initializes the pool with the serialized prefab and count
        /// </summary>
        protected void InitPool()
        {
            for (int i = 0; i < _initialCount; i++)
            {
                T obj = Instantiate(_objectPrefab, transform);
                m_poolQueue.Enqueue(obj);
                obj.gameObject.SetActive(false);
            }
        }

        /// <summary>
        /// Can be used to re-initialize the pool with different prefab and count values
        /// </summary>
        /// <param name="prefab"></param>
        /// <param name="count"></param>
        protected void InitPool(T prefab, int count)
        {
            _objectPrefab = prefab;
            _initialCount = count;
            InitPool();
        }

        /// <summary>
        /// Selects an item, returns and removes it from the pool. CAREFUL: Doesn't expands the pool if it's empty.
        /// </summary>
        /// <returns>An item form the pool</returns>
        public T Get()
        {
            T obj = m_poolQueue.Dequeue();
            obj.gameObject.SetActive(true);
            m_poolQueue.Enqueue(obj);

            return obj;
        }

        /// <summary>
        /// Selects an item, returns and removes it from the pool, places it at the given world position.
        /// CAREFUL: Doesn't expands the pool if it's empty.
        /// </summary>
        /// <param name="position"></param>
        /// <returns></returns>
        public T Get(Vector3 position)
        {
            T obj = m_poolQueue.Dequeue();
            obj.transform.position = position;
            obj.gameObject.SetActive(true);
            m_poolQueue.Enqueue(obj);

            return obj;
        }

        /// <summary>
        /// Returns an item to the pool, doesn't disables the GameObject.
        /// </summary>
        /// <param name="obj">Item to return</param>
        public void Return(T obj)
        {
            obj.gameObject.SetActive(false);
            m_poolQueue.Enqueue(obj);
        }
    }
}
```


```csharp
﻿using System.Collections.Generic;
using UnityEngine;

namespace FE.ObjectPool
{
    public class ObjectPool<T> where T : MonoBehaviour
    {
        private readonly int expand_step = 10;

        private readonly Queue<T> m_items;
        
        private readonly Transform m_parent;

        private readonly T m_prefab;

        public ObjectPool(T prefab, int count)
        {
            m_items = new Queue<T>(count);
            m_prefab = prefab;
            InitializePool(prefab, count);
        }

        public ObjectPool(T prefab, int count, Transform parent)
        {
            m_items = new Queue<T>(count);
            m_prefab = prefab;
            m_parent = parent;
            InitializePool(prefab, count, parent);
        }
        public ObjectPool(T prefab, int count, int expandStep, Transform parent)
        {
            m_items = new Queue<T>(count);
            expand_step = expandStep;
            m_prefab = prefab;
            m_parent = parent;
            InitializePool(prefab, count, parent);
        }

        public T Get()
        {
            if (m_items.Count == 0)
            {
                int c = 0;
                if (expand_step == 0)
                {
                    Debug.LogWarning("Expansion step is 0, pool is empty!");
                }
                else
                {
                    while (c < expand_step)
                    {
                        InstantiateInstance(m_prefab, m_parent);
                        c++;
                    }
                }
            }

            return m_items.Dequeue();
        }

        public void Return(T t)
        {
            m_items.Enqueue(t);
        }

        private void InitializePool(T prefab, int count)
        {
            for (int i = 0; i < count; i++) InstantiateInstance(prefab);
        }

        private void InitializePool(T prefab, int count, Transform parent)
        {
            for (int i = 0; i < count; i++) InstantiateInstance(prefab, parent);
        }

        private void InstantiateInstance(T prefab)
        {
            T instance = Object.Instantiate(prefab);
            m_items.Enqueue(instance);
            instance.gameObject.SetActive(false);
        }

        private void InstantiateInstance(T prefab, Transform parent)
        {
            T instance = Object.Instantiate(prefab, parent, true);
            m_items.Enqueue(instance);
            instance.gameObject.SetActive(false);
        }
    }
}
```
