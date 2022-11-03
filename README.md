### OpenWorldGame

#### 포톤 기반 오픈월드 게임 

#### PlayerController.cs

```C#
using System;
using System.Collections;
using System.Collections.Generic;
using Photon.Pun;
using Unity.VisualScripting;
using UnityEngine;
using UnityEngine.InputSystem;
using Random = UnityEngine.Random;

[RequireComponent(typeof(Rigidbody))]
public class PlayerController : MonoBehaviourPunCallbacks
{
    [SerializeField] float _speed = 100f;
    [SerializeField] float _radius = 1f;
    [SerializeField] float _jumpPower = 0.4f;
    [SerializeField] float _sprintSpeed = 200f;
    // TODO : 나중에 SerializeField 지우기
    [SerializeField] bool _isGrounded;
    [SerializeField] bool _isSprint;
    [SerializeField] Transform _camTrans;
    [SerializeField] int _damage=10;
    [SerializeField] float _moveAnimationMod = 200f;
    [SerializeField] float _jumpTime = 1f;
    [SerializeField] float _jumpTimer = 0f;
    int _animXSpeed;
    int _animYSpeed;
    int _animJump;
    int _animAttack;
    float _offsetSpeed;
    bool _canJump = true;
    Vector3 _direction;
    Rigidbody _rigidbody;
    Vector3 _moveDir;
    Animator _animator;
    AnimationEventHandler _animationEventHandler;
    Transform _playerBody;
    PhotonView _pv;

    public bool IsEquip;

    void Start()
    {
        Debug.Log("플레이어 생성 !");
        Init();
        AnimInit();
    }
    
    void Init()
    {
        _pv = GetComponent<PhotonView>();
        _animator = GetComponentInChildren<Animator>();
        _rigidbody = GetComponent<Rigidbody>();
        _offsetSpeed = _speed;
        _speed = 0;
        _playerBody = _animator.transform;
        _animationEventHandler = GetComponentInChildren<AnimationEventHandler>();
        _animationEventHandler.AttackEvent += AttackEvent;
    }

    void AnimInit()
    {
        _animXSpeed = Animator.StringToHash("xSpeed");
        _animYSpeed = Animator.StringToHash("ySpeed");
        _animAttack = Animator.StringToHash("randomAttack");
        _animJump = Animator.StringToHash("isJump");
    }

    void Update()
    {
        if (!_pv.IsMine)
            return;
        JumpDelayTimer();
    }



    void FixedUpdate()
    {
        if (!_pv.IsMine)
            return;
        Move();
        CheckGround();
    }

    void OnMove(InputValue value)
    {
        Vector2 inputMove = value.Get<Vector2>();
        _direction.x = inputMove.x;
        _direction.z = inputMove.y;
        _speed = Mathf.Approximately(_direction.magnitude, 0f) ? 0f : _offsetSpeed;
    }

    void OnJump()
    {
        if (!_isGrounded || !_canJump)
            return;
        Vector3 forceVel = _rigidbody.velocity;
        forceVel.y = -_jumpPower * Physics.gravity.y;
        _rigidbody.velocity = forceVel;
        _animator.SetBool(_animJump, true);
        _jumpTimer = _jumpTime;
    }

    void OnSprint(InputValue value)
    {
        _isSprint = value.isPressed;
    }

    void OnFire()
    {
        if (!_pv.IsMine)
            return;
        int randomAttack = Random.Range(1, 3);
        if (!_animator.GetCurrentAnimatorStateInfo(1).IsName("RandomAttack"))
        {
            _animator.SetFloat(_animAttack, randomAttack);
        }
    }

    void OnDrawGizmos()
    {
        Gizmos.DrawSphere(transform.position, _radius);
        Gizmos.DrawRay(transform.position + Vector3.up * 0.9f, transform.forward);
    }

    

    void JumpDelayTimer()
    {
        
        if (_jumpTimer <= 0.0f)
        {
            _canJump = true;
            _animator.SetBool(_animJump, false);
        }

        if (_jumpTimer > 0.0f)
        {
            _jumpTimer -= Time.deltaTime;
            _canJump = false;
        }
    }

    void Move()
    {
        
        float targetSpeed = _isSprint ? _sprintSpeed : _speed;
        float fallSpeed = _rigidbody.velocity.y;
        if (_direction.sqrMagnitude != 0)
        {
            Vector3 lookForward = new Vector3(_camTrans.forward.x, 0f, _camTrans.forward.z).normalized;
            Vector3 lookRight = new Vector3(_camTrans.right.x, 0f, _camTrans.right.z).normalized;
            //_moveDir = transform.TransformDirection(_direction).normalized * (targetSpeed * Time.deltaTime);
            _moveDir = (_direction.z * lookForward + _direction.x * lookRight) * (targetSpeed * Time.deltaTime);
            // 방향 일치시켜주기 - y를 안 넣는 이유는 앞으로 기울어지기 때문
            // transform.forward = lookForward로 하면 카메라에 따라 캐릭터도 회전하게 됨
            _playerBody.forward = _moveDir;
            _moveDir.y = fallSpeed;
            _rigidbody.velocity = _moveDir;
        }

        // Vector3 heading = _camera.localRotation * _direction;
        // transform.rotation = Quaternion.LookRotation(_direction);

        MoveAnimation(targetSpeed);
    }

    void MoveAnimation(float targetSpeed)
    {
        // 애니메이션
        float animSpeedY = _direction.z * targetSpeed / _moveAnimationMod;
        float animSpeedX = _direction.x * _speed / _moveAnimationMod;
        _animator.SetFloat(_animYSpeed, animSpeedY);
        _animator.SetFloat(_animXSpeed, animSpeedX);
    }

    void CheckGround()
    {
        if (Physics.CheckSphere(transform.position, _radius, 1 << 6, QueryTriggerInteraction.Ignore))
        {
            _isGrounded = true;
        }
        else
        {
            _isGrounded = false;
        }
    }

    public void AttackEvent(int damage)
    {
        
        RaycastHit hit;
        _animator.SetFloat(_animAttack, 0);
        Debug.DrawRay(transform.position + Vector3.up * 1.5f, transform.forward * 2);

        if (!Physics.Raycast(transform.position + Vector3.up * 0.9f, transform.forward, out hit, 1f))
            return;
        // TODO 디버그 삭제
        Debug.Log("Player Attack!");
        IDamageable damageable = hit.collider.GetComponent<IDamageable>();
        if (damageable.Equals(null))
            return;
        damageable.Damage(damage);
    }
}

```

